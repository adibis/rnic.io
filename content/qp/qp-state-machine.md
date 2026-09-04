---
title: "QP State Machine"
weight: 5
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "Every QP state transition is a contract: software supplies a fixed set of attributes, hardware refuses the transition if even one is missing. Reliable Connection's contract is the strictest of the three transports it's enforced against, and that's not incidental -- it's every parameter the reliability engine needs, collected at the one moment hardware can still refuse to proceed without it."
prev: /qp/installing-a-qp
next: /qp/single-qp-error-handling
---

## Locator

The master diagram with the QP Table / QP Cache block highlighted — the state machine described in this chapter is logic owned by that block, not a separate structure of its own.

<!-- IMAGE: Master RNIC block diagram (see docs/MASTER_DIAGRAM_INVENTORY.md), full canvas, all blocks present at reduced opacity (0.35) except QP Table and QP Cache, which are drawn at full opacity with the indigo highlight override (fill #eef2ff, stroke #4338ca at stroke-width 2, label text #312e81) per docs/SVG_STYLE.md's highlight/dim convention -- not their normal amber "shared/stateful resource" role color, which only applies at reduced opacity like every other dimmed block. Connector arrows between QP Table/Cache and the TX Scheduler, RX Parser/Reassembly, and Eviction Engine satellite block are at full opacity; every other arrow inherits the 0.35 dimming of its endpoints. -->

## Overview

A Queue Pair's state is not a status flag hardware happens to expose — it's a gate. RESET, INIT, RTR, RTS, and ERROR are the states a QP can carry live traffic through, in exactly that order forward, and every forward transition is a specific `ibv_modify_qp` call that hardware validates against a fixed, transport-dependent list of required attributes before it lets the QP move at all. Supply an incomplete set and the transition is refused, not partially applied. This chapter covers that state machine for Reliable Connection (RC) — the transport every later Part's QP is assumed to be running in — and, briefly, why RC's version of it carries more hardware-enforced state than the other two transports do.

## Block diagram

A zoomed state diagram: RESET, INIT, RTR, RTS as the core forward path, SQD as an RTS-reachable sub-graph with its own draining/drained internal states, ERROR reachable from any state, and RESET as ERROR's only way out.

<!-- IMAGE: State diagram, five/six primary nodes (RESET, INIT, RTR, RTS, ERROR, plus SQD as a satellite state attached to RTS) laid out left to right in forward-transition order. Solid arrows for the core RESET→INIT→RTR→RTS path, each labeled with its required ibv_modify_qp attribute mask bits (short form). A dashed bidirectional arrow between RTS and SQD, itself containing two internal sub-states (draining, drained) shown as a small nested pair. Dashed arrows from every state converging into ERROR, colored in the red "fault" role color per docs/SVG_STYLE.md. One arrow from ERROR back to RESET, labeled "only path out," full opacity, distinct from the dashed fault-arrows feeding in. -->

## Deep walkthrough

**The core path, and what each transition demands.** RESET→INIT requires `IBV_QP_STATE`, `IBV_QP_PKEY_INDEX`, `IBV_QP_PORT`, `IBV_QP_ACCESS_FLAGS` — enough to place the QP on a port and declare what remote operations it will accept, nothing about a peer yet. INIT→RTR requires `IBV_QP_STATE`, `IBV_QP_AV` (the address vector — the path to the peer), `IBV_QP_PATH_MTU`, `IBV_QP_DEST_QPN`, `IBV_QP_RQ_PSN`, `IBV_QP_MAX_DEST_RD_ATOMIC`, `IBV_QP_MIN_RNR_TIMER` — this is the transition that names a specific peer and sets the QP up to receive from it. RTR→RTS requires `IBV_QP_STATE`, `IBV_QP_SQ_PSN`, `IBV_QP_MAX_QP_RD_ATOMIC`, `IBV_QP_RETRY_CNT`, `IBV_QP_RNR_RETRY`, `IBV_QP_TIMEOUT` — the transition that arms the QP to send. Miss one required bit in the mask on any of these calls and hardware refuses the transition outright; there is no partially-applied state to clean up afterward, because the QP never left its prior state in the first place.

Line up RTR→RTS's required set against what UC and UD ask for at the same transition and the pattern is exact: UC needs only `IBV_QP_STATE` and `IBV_QP_SQ_PSN`; UD needs the same two. Neither carries `IBV_QP_MAX_QP_RD_ATOMIC`, `IBV_QP_RETRY_CNT`, `IBV_QP_RNR_RETRY`, or `IBV_QP_TIMEOUT` — every one of RC's extra fields is a parameter Part 5's reliability engine needs to run its own retry state machine, and UC/UD don't have one to configure. This isn't RC being more thoroughly specified for its own sake. It's hardware collecting, at the one moment it can still refuse to proceed without them, every input a per-QP retry timer and outstanding-request counter will need for the entire lifetime of the connection that follows. INIT→RTR shows the same shape in miniature: RC alone carries `IBV_QP_MAX_DEST_RD_ATOMIC` and `IBV_QP_MIN_RNR_TIMER`, both meaningless to a transport with no responder-side read/atomic resource limit and no RNR-NAK retry policy to configure.

**SQD: a clean way to stop without an error.** RTS is not a dead end software has to tear down and rebuild from just to change a live QP's configuration. RTS→SQD (Send Queue Drain) stops the QP from accepting *new* posted sends while letting already-posted ones finish and their responses arrive — internally, SQD has its own draining (still finishing in-flight work) and drained (fully quiesced) sub-states, and if `en_sqd_async_notify` was set on entry, `IBV_EVENT_SQ_DRAINED` fires exactly on the draining→drained internal transition. Once that event lands, software can safely modify send-side attributes and transition back RTS→SQD→RTS. This is the mechanism that makes SQD categorically different from ERROR below: SQD is a deliberate, software-requested, clean quiesce with a defined completion signal; ERROR is not requested, has no drain-to-completion signal, and is not reversible in place. A drain can itself be interrupted by a transition into SQE or ERROR instead of completing normally — a clean stop can still be overtaken by a real fault.

**ERROR: where RC and the other two transports genuinely diverge.** On a completion error in the send queue, UD and UC move only the send queue to a distinct SQE (Send Queue Error) state — the QP stays alive, the receive queue is untouched, and software can recover it directly back to RTS without a full rebuild, because the fault is scoped to exactly the part of the QP that produced it. RC has no SQE state at all. Any completion error on an RC QP takes the *entire* QP to ERROR, both queues, no partial-recovery path. The reason traces directly back to what RTR→RTS just configured: RC's whole contract is ordered, reliable, acknowledged delivery, backed by a PSN sequence and a retry state machine tracking exactly what's been sent and what's been acknowledged. A fault serious enough to reach a completion doesn't just affect one queue's next operation — it puts that sequencing state itself in doubt, and hardware has no way to certify that only the send side's sequencing is unsound while the receive side's is still trustworthy. So it doesn't try. What actually happens to a QP's outstanding work once it lands in ERROR — and the isolation guarantee that gives other QPs — is the next chapter's subject.

**Out of ERROR, there is exactly one door.** No transition takes a QP directly from ERROR back to RTS. The only legal move is ERROR→RESET, after which the QP is rebuilt from scratch through the same INIT→RTR→RTS sequence described above, supplying every required attribute again as if the QP were new. What "cleared" actually means for state that was in flight at the moment of the reset — replay-buffer contents, partially-received messages, DMA already underway — is genuinely underdocumented territory this book takes on directly in [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) rather than glossing over it here.

> **Design rationale.** Hardware could, in principle, let software assemble a QP's configuration incrementally and only check completeness lazily, the first time the QP actually needs an attribute nobody set. It doesn't, because that would turn a configuration bug into a wire-visible failure — a QP silently sending with a garbage or zero-valued retry timeout, say — discovered only once real traffic depends on it, instead of a rejected `ibv_modify_qp` call the driver can report immediately. Enforcing the attribute mask at the transition itself moves every one of those failures to the earliest possible moment, before a single packet carrying that QP's state has gone anywhere. The cost is a rigid, transport-specific mask table baked into the silicon's validation logic rather than a more flexible "fill in whatever, whenever" model — a trade rnic.io's per-Part checklist keeps making in different forms: catch it at the gate, not downstream.

## Interface and timing reference table

| Field | Set at | RC scope | Meaning |
|---|---|---|---|
| `path_mtu` | INIT→RTR | RC/UC only | Path MTU negotiated for this connection |
| `dest_qp_num` | INIT→RTR | RC/UC only | Destination QP number this QP is bound to |
| `rq_psn` | INIT→RTR | RC/UC only | Next PSN this QP expects to receive |
| `max_dest_rd_atomic` | INIT→RTR | RC only | Responder resources: outstanding RDMA read/atomic this QP will accept as responder |
| `min_rnr_timer` | INIT→RTR | RC only | Minimum RNR NAK timer this QP will report to a peer |
| `sq_psn` | RTR→RTS | all | Next PSN this QP will send |
| `max_rd_atomic` | RTR→RTS | RC only | Requester resources: outstanding RDMA read/atomic this QP may have in flight as requester |
| `timeout` | RTR→RTS | RC only | Local ACK timeout, primary path |
| `retry_cnt` | RTR→RTS | RC only | Retry count before `IBV_WC_RETRY_EXC_ERR` |
| `rnr_retry` | RTR→RTS | RC only | RNR retry count before `IBV_WC_RNR_RETRY_EXC_ERR` |
| `en_sqd_async_notify` | any, before SQD | all | Enables `IBV_EVENT_SQ_DRAINED` on the draining→drained transition |

That's the state machine's full input surface — the reference to come back to once a later chapter starts asking what happens when one of these fields is wrong at runtime rather than missing at setup. [Single-QP Error Handling](../single-qp-error-handling) is exactly that chapter: what a QP in ERROR actually does to its outstanding work, and the isolation guarantee that keeps one QP's fault from touching any other.
