---
title: "QP Context"
weight: 1
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "Two fields on every RC Queue Pair count outstanding RDMA reads and atomics -- one for what this QP has sent, one for what it will accept -- and confusing them is the single most common misreading of a QP's context. Here's the full field list, wire-format ties included, and which fields hardware updates on its own without a single software write."
next: /qp/the-qp-table
---

[How to Read This Guide](/overview/how-to-read-this-guide) named the checklist this Part runs in full, and [One Transaction, Start to Finish](/overview/one-transaction-start-to-finish) already touched this chapter's subject twice without stopping to open it: once to say the sending adapter needs "the sending QP's live context" before it can build a packet, once to say the receiving side performs "the same kind of QP Table/Cache lookup" on arrival. This is where that context actually gets defined, field by field, before the next two chapters cover where it's stored and what gets cached.

## Overview

A CPU core's architectural state -- program counter, registers, in-flight transaction trackers -- is scoped to one thread of execution; a Queue Pair is the RNIC's own equivalent, scoped instead to one specific peer connection. A Queue Pair's context is the complete set of state hardware needs to send and receive on that connection correctly: which state the QP is in, who it's connected to, what sequence numbers it's tracking, and how many outstanding operations it's allowed to carry in either direction. Software supplies most of it once, at specific state transitions; hardware updates the rest continuously, on every packet, without a software write in the loop.

## Locator

![The master RNIC block diagram with the QP Table and QP Cache pair highlighted and every other block dimmed -- this chapter covers the data living inside one entry of that pair, not the structure itself.](/images/qp/01-qp-context-locator.svg)

Every field below lives inside one entry of [The QP Table](/qp/the-qp-table), which this Part covers next -- this chapter is about what's in that entry, not the structure holding it.

## The fields, and what's really two roles wearing one name

The most reliable public mirror of a QP's context is `ibv_qp_attr`, the struct the Linux verbs API (`ibv_modify_qp`) uses to read and write it. It is a software-facing API structure, not a description of the silicon-side layout an RNIC actually keeps on-chip or in its backing table -- real hardware is free to pack, reorder, or split these fields however its pipeline needs. But every field IBTA's QP context defines has a corresponding `ibv_qp_attr` field, and the verbs struct is real, open-source, and directly readable, which makes it the right reference point for what a QP context contains even before asking how a specific chip stores it.

Two fields are worth pulling out before the full table, because they're the most commonly misread pair in this entire context:

- **`max_rd_atomic`** -- how many RDMA read and atomic requests *this* QP may have outstanding **as the requester**, at once. It bounds how deep this QP's own send-side read/atomic pipeline can run.
- **`max_dest_rd_atomic`** -- how many incoming RDMA read and atomic requests *this* QP will accept and hold **as the responder**, before it has to push back. It bounds how many of the remote peer's read/atomic requests this QP can have in flight against it.

These are independently negotiated, apply to opposite directions of the same connection, and are very often asymmetric -- a QP that mostly issues reads and rarely serves them might run a high `max_rd_atomic` and a low `max_dest_rd_atomic`. Reading one where the other belongs is the single easiest way to misdiagnose a QP context.

Beyond that pair, the fields split cleanly by transport scope. UD (Unreliable Datagram) has no remote-access permission concept at all -- it supports only SEND/RECEIVE, so `qp_access_flags` doesn't apply to it either, alongside the rest of RC/UC's connection-oriented fields below. RC (Reliable Connection) carries every field UC does plus the ones only a connection with hardware-managed reliability needs: `timeout`, `retry_cnt`, `rnr_retry`, `min_rnr_timer`, and the `max_rd_atomic`/`max_dest_rd_atomic` pair above. UD (Unreliable Datagram) carries none of that -- no reliability state to speak of, no `dest_qp_num` even, since a UD QP names its destination per-message rather than once at connection setup. UC (Unreliable Connected) sits between the two: connected like RC, but without the reliability fields, since there's no retry engine reading them. (A fourth transport, XRC, decouples a QP's receive side from any one specific send-side QP further still, to cut per-peer QP count in large fan-in deployments -- it's out of scope for this Part, which assumes RC throughout.)

## Block diagram

![One QP Context entry as a struct-layout diagram: identity/connection fields, permissions, the max_rd_atomic/max_dest_rd_atomic pair drawn with explicit directional arrows toward the requester and responder sides respectively, RC-only reliability fields set apart by a dashed border, and the two live hardware-updated PSN counters highlighted in indigo.](/images/qp/01-qp-context-block.svg)

This chapter's one diagram earns its place on exactly the pair the prose above spends its first section on: `max_rd_atomic` and `max_dest_rd_atomic` drawn with their directions made explicit, since the two fields differing only in *which side of the connection* is counted is precisely what makes them easy to transpose.

## Set once, or updated by hardware on every packet

Most of a QP context is written by software once, at a specific state transition, and then held fixed by hardware until the next explicit write: `path_mtu`, `dest_qp_num`, the address vectors, `pkey_index`, `max_rd_atomic`, `max_dest_rd_atomic`, `min_rnr_timer`, `timeout`, `retry_cnt`, `rnr_retry`, `port_num`, `qkey`. [Installing a QP](/qp/installing-a-qp) and [QP State Machine](/qp/qp-state-machine) cover exactly which fields each transition requires and why hardware refuses the transition without them. `qp_access_flags` is a third case, distinct from either bucket: it must first be supplied at RESET→INIT like the fields above, but -- unlike `pkey_index` or `port_num` -- it isn't frozen there for the QP's whole life. [Installing a QP](/qp/installing-a-qp) covers the live, same-state path that re-opens it later.

A smaller set moves continuously, under hardware's own control, with no software write involved at all: `rq_psn` (the next PSN this QP's receive side expects, advancing on each in-order accepted packet) and `sq_psn` (the next PSN this QP's send side will stamp on a new message, advancing on each one sent). A live count of outstanding requester- and responder-side read/atomic operations is tracked against the `max_rd_atomic`/`max_dest_rd_atomic` ceilings above, but the verbs layer only exposes the ceiling, not the running counter -- how a given implementation names or organizes that counter internally isn't something this book states as fact. `sq_draining` is a third, narrower case: a status bit hardware sets, software only ever reads.

## Deep walkthrough: three fields worth tracing all the way through

**`max_dest_rd_atomic`** -- the responder-side ceiling from above.
- *What*: an 8-bit value, RC only, set once at the INIT→RTR transition.
- *Where this is checked*: at the responder, against a live count of that QP's currently outstanding incoming read/atomic requests, whenever a new `RDMA_READ_REQUEST` or atomic opcode arrives for it.
- *When it fails*: a request that would push the live count past this ceiling isn't treated as a transient, retry-and-move-on condition -- it's classified as a QP access error (the same event class that covers an `rkey` violation), and it forces the QP straight to the ERROR state.
- *What happens next*: [Single-QP Error Handling](/qp/single-qp-error-handling) owns everything downstream of that -- what "forced to ERROR" actually does to outstanding work on both queues.

**`dest_qp_num`** -- the peer's QP number, RC/UC only, set once at INIT→RTR, held fixed for the connection's life.
- *What*: a 32-bit field holding the 24-bit destination QP number -- the exact width and position [What Is RoCEv2](/overview/what-is-rocev2) already described inside the BTH.
- *Where this is used*: not checked against an incoming packet so much as stamped into every outgoing one -- the TX pipeline (Part 3 (TX Pipeline)) builds each packet's BTH by copying this QP's `dest_qp_num` straight into the destination-QP field, on every single packet this QP ever sends.
- *When it's wrong*: it isn't something that can fail at runtime -- it's fixed at INIT→RTR and only reachable again through a full RESET. A QP simply cannot send to a different peer without being torn down and rebuilt.
- *What happens next*: nothing to retry here; this field's only failure mode is a software bug at connection setup, not a hardware fault.

**`rq_psn` / `sq_psn`** -- the receive- and send-side sequence counters.
- *What*: 32-bit fields, RC/UC only for `rq_psn`, all transports for `sq_psn`. `rq_psn` starts at a value software supplies at INIT→RTR (the PSN the peer's first packet is expected to carry); `sq_psn` starts at a value software supplies at RTR→RTS, then advances under hardware's own control from there.
- *Where this is checked*: at the receiving QP, comparing `rq_psn` against the PSN carried in every arriving packet's BTH -- exactly which pipeline stage performs that comparison, and what happens to a packet that doesn't match, is the RX pipeline's (Part 4 (RX Pipeline)) open question, deliberately not resolved here (see [One Transaction, Start to Finish](/overview/one-transaction-start-to-finish) for the walkthrough that first flags it).
- *What happens next*: a PSN mismatch is reliability-engine (Part 5 (Reliability Engine)) territory -- this chapter's job stops at naming the field and what it holds, not how a gap or duplicate gets handled.

## Design rationale

> **Design rationale.** Splitting `max_rd_atomic` and `max_dest_rd_atomic` into two independent fields, rather than one shared "outstanding read/atomic" limit, is what lets a connection be asymmetric on purpose -- a storage target that mostly serves reads and rarely issues them can advertise a generous `max_dest_rd_atomic` and a minimal `max_rd_atomic`, sized to its actual role, instead of both ends negotiating down to the more conservative of two roles neither of them is symmetrically playing. The cost of that flexibility is exactly the confusion this chapter opened with: two fields that sound like they mean the same thing, differing only in whose queue is counted, is a genuinely easy pair to transpose, and getting it backwards doesn't fail loudly at setup -- it just silently under- or over-provisions one direction of a connection until traffic patterns expose it.

[The QP Table](/qp/the-qp-table) picks this context up as a single addressable entry and covers where it actually lives.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
