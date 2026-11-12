---
title: "Single-QP Error Handling"
weight: 6
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "A QP in ERROR doesn't fail loudly and stop -- it fails loudly and keeps going, flushing every queued work request through the completion queue with a status that says none of this actually happened. Here's what that flush actually does, and why the QP sitting next to it never even notices."
prev: /qp/qp-state-machine
next: /qp/multi-qp-error-arbitration
metaDescription: "What happens when an RDMA QP goes to ERROR: every outstanding Work Request is flushed with a completion, while neighbouring QPs carry on."
seoTitle: "RDMA QP Error Handling and Flush"
---

## Overview

A QP that reaches ERROR does not stop processing — it starts flushing.

> [!WARNING] Analogy
> A CPU exception or branch misprediction doesn't strand the instructions still in flight behind it in limbo -- it squashes them, in order, each with a defined outcome, rather than letting them vanish silently or complete against a state that's no longer valid. A QP in ERROR does the same thing for its own in-flight work. Every work request already posted and not yet completed, and every work request posted afterward, drains through the completion queue carrying a status that says the operation never happened, until nothing is left in either queue. Nothing about this touches any other QP on the same adapter: ERROR is scoped to exactly the QP that faulted, by construction, and that isolation guarantee is this chapter's real subject.

## Locator

![The master RNIC block diagram with the QP Table and QP Cache pair highlighted, the same target as the QP State Machine chapter -- this chapter covers the fault path through that same block.](/images/qp/06-single-qp-error-handling-locator.svg)

[QP State Machine](../qp-state-machine) covered how a QP reaches ERROR and named this chapter as the place that explains what ERROR actually does to a QP's outstanding work. This is that chapter — for exactly one QP, in isolation. [Multi-QP Error Arbitration](../multi-qp-error-arbitration) picks up immediately after with the harder question of many QPs faulting together.

## Block diagram

![One QP's send and receive queues both draining into the Completion Queue, every entry stamped with a flush-in-error completion status, inside a fault isolation boundary that excludes the CQ itself and any other QP.](/images/qp/06-single-qp-error-handling-block1.svg)

## Deep walkthrough

**What a flush actually is.** Once a QP is in ERROR, hardware doesn't discard outstanding work silently — it completes every one of it, in order, with a flush-in-error completion status instead of a normal success status. This includes work already posted and awaiting processing when the fault occurred, and work posted afterward, for as long as the application keeps posting to a QP it hasn't yet noticed is dead. A flush is not a hardware fault of its own; it's the QP telling software, one completion at a time, "this didn't happen, and neither will anything you post from here." Draining continues until both queues are empty, at which point the QP is fully quiesced in ERROR and waiting for a RESET.

**Two kinds of fault, and why the distinction matters.** Be precise about which layer each name below actually belongs to: the underlying condition -- a transport-layer violation in an arriving packet, or a permission/bounds/resource violation on an operation -- is wire-level and standard-defined, the same AETH-syndrome territory [Permission and Bounds Violations](/mr/permission-and-bounds-violations) covers in depth; the two local fault-event classes named below are how each condition surfaces locally, as an asynchronous event on the responder's own QP, not the origin of the classification itself. Not every path into ERROR means the same thing about what went wrong. An **invalid-request error** (the verbs event `IBV_EVENT_QP_REQ_ERR`, where "REQ" names the incoming request, not the requester side) fires when this QP, acting as responder, detects a transport-layer violation in a request arriving for it — an unsupported or out-of-sequence opcode. It's RC-only, and is rare enough in practice to be a sign of a problem in the subnet, or an RDMA device sending illegal packets — something is wrong with what's arriving, not with how this QP was set up. An **access-violation error** is a different animal entirely: a misaligned atomic request, more outstanding RDMA read/atomic requests than the responder-side ceiling allows, a remote-key violation, or a length error on an operation without immediate data. This class usually reflects a bug in the requesting application's own usage, not the network or the peer doing anything wrong. Both auto-transition the QP to ERROR identically from hardware's side -- confirmed directly, not just observed as a pattern: IBTA-V1's Table 58 (Responder Error Behavior Summary) maps every condition named in both groups above, for RC, to the identical fault class (Responder Class C -- current WQE completed in error, subsequent WQEs flushed, queue to error state). The distinction this chapter draws between the two groups is entirely about where a diagnosing engineer should look afterward, protocol/wire problem versus local-programming problem, not about how the QP itself behaves once it's there -- the standard itself draws no behavioral line between them at all.

**Isolation is the property, not an accident.** Nothing in either error path touches a sibling QP on the same adapter. The flush behavior, the ERROR state itself, and the eventual RESET-to-recover path are all scoped to the one QP context that faulted — a second QP carrying on normal RTS traffic through the exact same TX/RX pipeline stages, cached in the exact same QP Cache, keeps running without observing anything. That isolation is precisely the property [Multi-QP Error Arbitration](../multi-qp-error-arbitration) complicates: isolation between two QPs faulting independently is one problem; many QPs faulting from one shared cause, all needing to report through the same channel at once, is a different one.

**A third trigger, and the one that doesn't stay single-QP.** A **QP-fatal event** is more than a generic "something prevented further completions" catch-all -- it has a cause the other two triggers don't have: an error on a Completion Queue or a Shared Receive Queue this QP is associated with. When a CQ or SRQ itself hits an error, *every* QP associated with it -- not just the one that happened to trigger the condition -- transitions to ERROR and fires its own QP-fatal event, simultaneously. The SRQ half of this is directly confirmed: IBTA-V1's Table 58 names "Local QP Error" (Responder Class A) explicitly as including the case where "a receive queue which is associated with a shared receive queue is unable to fetch a WQE from the shared receive queue due to an error condition in the shared receive queue." The CQ half -- a CQ error alone, with no SRQ involved -- isn't confirmed by that same table entry; it's sourced separately (see [CQ Overflow](/cq/cq-overflow), which documents one concrete way a CQ itself reaches an error condition that cascades outward this way). This chapter's isolation story still holds for the requester-side and access-violation triggers, both scoped to the one QP that faulted; the QP-fatal trigger via a shared CQ/SRQ is the one documented, concrete mechanism by which a single upstream cause takes out many QPs at the exact same instant. [Multi-QP Error Arbitration](../multi-qp-error-arbitration) is where that scenario actually gets reasoned through.

**SRQ-associated QPs need one more signal before teardown.** A QP that shares a Shared Receive Queue with other QPs can't simply be destroyed the moment it reaches ERROR — some of its outstanding receive-side work may still reference WQEs pulled from that shared queue, and destroying the QP before that work finishes draining risks leaving the SRQ itself in an inconsistent state for the QPs still using it. A dedicated **last-SRQ-WQE-consumed event** is the signal that closes this gap: it fires once this QP has consumed the last SRQ-sourced WQE it will ever consume, and it's the event software waits for before it's safe to tear the QP down without disturbing its SRQ siblings.

![An SRQ-associated QP in ERROR still draining WQEs it already pulled from a Shared Receive Queue two other healthy QPs also use, waiting for the last-SRQ-WQE-consumed event before it's safe to destroy, with the SRQ and its other QPs untouched throughout.](/images/qp/06-single-qp-error-handling-block2.svg)

{{< flow "qp-flush" >}}

> [!TIP] Why
> Flushing every outstanding operation with an explicit status, rather than simply halting the QP and leaving posted-but-unprocessed work in limbo, is what makes fault recovery a software-observable, boundedly-long process instead of an open question the application has to poll for indefinitely. It costs hardware a real obligation — walk both queues to completion, in order, generating a real CQE for each entry, even though none of them succeeded — instead of just setting a flag and stopping. That cost buys the application a guarantee: post-then-poll always terminates, one way or another, for every request it ever issued, fault or no fault. The alternative (silently dropping queued work with no completion at all) would turn every fault into a hang the application has no event to wake up on.

One QP faulting is a contained, well-defined event. [Multi-QP Error Arbitration](../multi-qp-error-arbitration) asks what happens when the isolation boundary this chapter just described has to hold under many simultaneous faults instead of one.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- Table 58 (Responder Error Behavior Summary) confirms this chapter's requester-side-transport-error and access-violation-error groups both map to the identical fault class (Responder Class C, RC) despite their different underlying causes, and its "Local QP Error" row (Responder Class A) directly names the SRQ-WQE-fetch-failure case this chapter's QP-fatal trigger describes.
