---
title: "Single-QP Error Handling"
weight: 6
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "A QP in ERROR doesn't fail loudly and stop -- it fails loudly and keeps going, flushing every queued work request through the completion queue with a status that says none of this actually happened. Here's what that flush actually does, and why the QP sitting next to it never even notices."
prev: /qp/qp-state-machine
next: /qp/multi-qp-error-arbitration
---

## Locator

![The master RNIC block diagram with the QP Table and QP Cache pair highlighted, the same target as the QP State Machine chapter -- this chapter covers the fault path through that same block.](/images/qp/06-single-qp-error-handling-locator.svg)

[QP State Machine](../qp-state-machine) covered how a QP reaches ERROR and named this chapter as the place that explains what ERROR actually does to a QP's outstanding work. This is that chapter — for exactly one QP, in isolation. [Multi-QP Error Arbitration](../multi-qp-error-arbitration) picks up immediately after with the harder question of many QPs faulting together.

## Overview

A CPU exception or branch misprediction doesn't strand the instructions still in flight behind it in limbo -- it squashes them, in order, each with a defined outcome, rather than letting them vanish silently or complete against a state that's no longer valid. A QP that reaches ERROR does the same thing for its own in-flight work: it does not stop processing — it starts flushing. Every work request already posted and not yet completed, and every work request posted afterward, drains through the completion queue carrying a status that says the operation never happened, until nothing is left in either queue. Nothing about this touches any other QP on the same adapter: ERROR is scoped to exactly the QP that faulted, by construction, and that isolation guarantee is this chapter's real subject.

## Block diagram

![One QP's send and receive queues both draining into the Completion Queue, every entry stamped IBV_WC_WR_FLUSH_ERR, inside a fault isolation boundary that excludes the CQ itself and any other QP.](/images/qp/06-single-qp-error-handling-block1.svg)

## Deep walkthrough

**What a flush actually is.** Once a QP is in ERROR, hardware doesn't discard outstanding work silently — it completes every one of it, in order, with `IBV_WC_WR_FLUSH_ERR` instead of a normal success status. This includes work already posted and awaiting processing when the fault occurred, and work posted afterward, for as long as the application keeps posting to a QP it hasn't yet noticed is dead. A flush is not a hardware fault of its own; it's the QP telling software, one completion at a time, "this didn't happen, and neither will anything you post from here." Draining continues until both queues are empty, at which point the QP is fully quiesced in ERROR and waiting for a RESET.

**Two kinds of fault, and why the distinction matters.** Worth being precise about which layer each name below actually belongs to: the underlying condition -- a transport-layer violation in an arriving packet, or a permission/bounds/resource violation on an operation -- is wire-level and standard-defined, the same AETH-syndrome territory [Permission and Bounds Violations](/mr/permission-and-bounds-violations) covers in depth; `IBV_EVENT_QP_REQ_ERR` and `IBV_EVENT_QP_ACCESS_ERR` are simply the Linux/OFA verbs API's names for how each condition surfaces locally, as an asynchronous event on the responder's own QP, not the origin of the classification itself. Not every path into ERROR means the same thing about what went wrong. `IBV_EVENT_QP_REQ_ERR` fires when the responder side of this QP detects a transport-layer violation in a packet arriving for it — an unsupported or out-of-sequence opcode. It's RC-only, and NVIDIA's RDMA Aware Networks Programming User Manual calls it rare enough to be a sign of "problems in the subnet or when an RDMA device sends illegal packets" — something is wrong with what's arriving, not with how this QP was set up. `IBV_EVENT_QP_ACCESS_ERR` is a different animal entirely: a misaligned atomic request, more outstanding RDMA read/atomic requests than `max_dest_rd_atomic` allows, an R_Key violation, or a length error on an operation without immediate data. The same manual frames this plainly: these "usually occur because of bugs in the user code," not because the network or the peer did anything wrong. Both auto-transition the QP to ERROR identically from hardware's side — the distinction is entirely about where a diagnosing engineer should look afterward, protocol/wire problem versus local-programming problem, not about how the QP itself behaves once it's there.

**Isolation is the property, not an accident.** Nothing in either error path touches a sibling QP on the same adapter. The flush behavior, the ERROR state itself, and the eventual RESET-to-recover path are all scoped to the one QP context that faulted — a second QP carrying on normal RTS traffic through the exact same TX/RX pipeline stages, cached in the exact same QP Cache, keeps running without observing anything. That's worth stating explicitly here because it's precisely the property [Multi-QP Error Arbitration](../multi-qp-error-arbitration) complicates: isolation between two QPs faulting independently is one problem; many QPs faulting from one shared cause, all needing to report through the same channel at once, is a different one.

**A third trigger, and the one that doesn't stay single-QP.** `IBV_EVENT_QP_FATAL` is more than a generic "something prevented further completions" catch-all -- NVIDIA's RDMA Aware Networks Programming User Manual documents a cause for it that the other two triggers don't have: an error on a Completion Queue or a Shared Receive Queue this QP is associated with. When a CQ or SRQ itself hits an error, *every* QP associated with it -- not just the one that happened to trigger the condition -- transitions to ERROR and fires its own `IBV_EVENT_QP_FATAL`, simultaneously. This chapter's isolation story still holds for `QP_REQ_ERR` and `QP_ACCESS_ERR`, both scoped to the one QP that faulted; `QP_FATAL` via a shared CQ/SRQ is the one documented, concrete mechanism by which a single upstream cause takes out many QPs at the exact same instant. [Multi-QP Error Arbitration](../multi-qp-error-arbitration) is where that scenario actually gets reasoned through.

**SRQ-associated QPs need one more signal before teardown.** A QP that shares a Shared Receive Queue with other QPs can't simply be destroyed the moment it reaches ERROR — some of its outstanding receive-side work may still reference WQEs pulled from that shared queue, and destroying the QP before that work finishes draining risks leaving the SRQ itself in an inconsistent state for the QPs still using it. `IBV_EVENT_QP_LAST_WQE_REACHED` is the signal that closes this gap: it fires once this QP has consumed the last SRQ-sourced WQE it will ever consume, and it's the event software waits for before it's safe to tear the QP down without disturbing its SRQ siblings.

![An SRQ-associated QP in ERROR still draining WQEs it already pulled from a Shared Receive Queue two other healthy QPs also use, waiting for IBV_EVENT_QP_LAST_WQE_REACHED before it's safe to destroy, with the SRQ and its other QPs untouched throughout.](/images/qp/06-single-qp-error-handling-block2.svg)

> **Design rationale.** Flushing every outstanding operation with an explicit status, rather than simply halting the QP and leaving posted-but-unprocessed work in limbo, is what makes fault recovery a software-observable, boundedly-long process instead of an open question the application has to poll for indefinitely. It costs hardware a real obligation — walk both queues to completion, in order, generating a real CQE for each entry, even though none of them succeeded — instead of just setting a flag and stopping. That cost buys the application a guarantee: post-then-poll always terminates, one way or another, for every request it ever issued, fault or no fault. The alternative (silently dropping queued work with no completion at all) would turn every fault into a hang the application has no event to wake up on.

One QP faulting is a contained, well-defined event. [Multi-QP Error Arbitration](../multi-qp-error-arbitration) asks what happens when the isolation boundary this chapter just described has to hold under many simultaneous faults instead of one.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
- NVIDIA, *RDMA Aware Networks Programming User Manual*, "Events" section -- source for the direct quotes on `IBV_EVENT_QP_REQ_ERR` and `IBV_EVENT_QP_ACCESS_ERR` causes above. A vendor doc, not a standards document, per this book's citation convention (see docs/REFERENCES.md).
