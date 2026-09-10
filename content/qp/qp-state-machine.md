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

## Overview

A Queue Pair's state machine is a bring-up sequencer: each state gates exactly which operations are legal next, refusing to advance until its own prerequisites are satisfied. A Queue Pair's state is not a status flag hardware happens to expose — it's that same kind of gate.

> [!WARNING] Analogy
> The same shape as a core's reset FSM walking from reset-vector-fetch through to fetch-enabled, or a link protocol's LTSSM walking a physical link from Detect through Polling/Configuration to L0.

RESET, INIT, RTR, RTS, and ERROR are the states a QP can carry live traffic through, in exactly that order forward, and every forward transition is a specific state-transition request that hardware validates against a fixed, transport-dependent list of required attributes before it lets the QP move at all. Supply an incomplete set and the transition is refused, not partially applied. This chapter covers that state machine for Reliable Connection (RC) — the transport every later part's QP is assumed to be running in — and, briefly, why RC's version of it carries more hardware-enforced state than the other two transports do.

## Locator

![The master RNIC block diagram with the QP Table and QP Cache pair highlighted -- the state machine this chapter covers is logic owned by that block, not a separate structure.](/images/qp/05-qp-state-machine-locator.svg)

## Block diagram

A zoomed state diagram: RESET, INIT, RTR, RTS as the core forward path, SQD as an RTS-reachable sub-graph with its own draining/drained internal states, ERROR reachable from any state, and RESET as ERROR's only way out.

![The RC QP state machine: RESET through INIT, RTR, RTS labeled with each transition's required attribute mask, SQD as a satellite draining state off RTS, every state's fatal-error path converging into ERROR, and the single ERROR-to-RESET arrow as the only way out.](/images/qp/05-qp-state-machine-block.svg)

## Deep walkthrough

**The core path, and what each transition demands.** IBTA-V1 itself specifies, per transport type, exactly which QP attributes a state transition requires before hardware may permit it -- this mandatory/optional attribute list is the standard's own QP state-machine definition. RESET→INIT requires the state target itself, the partition-key index attribute, the port-binding attribute, and the remote-access permission mask — enough to place the QP on a port and declare what remote operations it will accept, nothing about a peer yet. INIT→RTR requires the state target, the address vector (the path to the peer), the path MTU attribute, the destination-QP field, the receive-side PSN counter's starting value, the responder-side outstanding-read/atomic ceiling, and the RNR-NAK timer floor — this is the transition that names a specific peer and sets the QP up to receive from it. RTR→RTS requires the state target, the send-side PSN counter's starting value, the requester-side outstanding-read/atomic ceiling, the retry-count ceiling, the RNR-retry ceiling, and the ACK timeout — the transition that arms the QP to send. Miss one required attribute on any of these calls and hardware refuses the transition outright; there is no partially-applied state to clean up afterward, because the QP never left its prior state in the first place.

Line up RTR→RTS's required set against what UC and UD ask for at the same transition and the pattern is exact: UC needs only the state target and the send-side PSN counter's starting value; UD needs the same two. Neither carries the requester-side outstanding-read/atomic ceiling, the retry-count ceiling, the RNR-retry ceiling, or the ACK timeout — every one of RC's extra fields is a parameter Part 5 (Reliability Engine) needs to run its own retry state machine, and UC/UD don't have one to configure. This isn't RC being more thoroughly specified for its own sake. It's hardware collecting, at the one moment it can still refuse to proceed without them, every input a per-QP retry timer and outstanding-request counter will need for the entire lifetime of the connection that follows. INIT→RTR shows the same shape in miniature: RC alone carries the responder-side outstanding-read/atomic ceiling and the RNR-NAK timer floor, both meaningless to a transport with no responder-side read/atomic resource limit and no RNR-NAK retry policy to configure.

**SQD: a clean way to stop without an error.** RTS is not a dead end software has to tear down and rebuild from just to change a live QP's configuration. RTS→SQD (Send Queue Drain) stops the QP from accepting *new* posted sends while letting already-posted ones finish and their responses arrive — internally, SQD has its own draining (still finishing in-flight work) and drained (fully quiesced) sub-states, and if an asynchronous-notification option was requested on entry, a drain-complete event fires exactly on the draining→drained internal transition. Once that event lands, software can safely modify send-side attributes and transition back RTS→SQD→RTS. This is the mechanism that makes SQD categorically different from ERROR below: SQD is a deliberate, software-requested, clean quiesce with a defined completion signal; ERROR is not requested, has no drain-to-completion signal, and is not reversible in place. For RC specifically, a drain can itself be interrupted by a transition straight into ERROR instead of completing normally — a clean stop can still be overtaken by a real fault. (UC and UD, which do have a distinct SQE state below, can have a drain interrupted into SQE instead; RC has no SQE state to interrupt into, which the next section covers directly.)

![RC's RTS state has exactly one fault exit, straight to ERROR with no partial recovery, while UC/UD's RTS can also reach a send-queue-only SQE state that recovers directly back to RTS without a rebuild -- alongside a summary of how many attributes each transport's RTR-to-RTS transition actually requires: six for RC, two for UC/UD.](/images/qp/05-qp-state-machine-transport-compare.svg)

**ERROR: where RC and the other two transports sharply diverge.** On a completion error in the send queue, UD and UC move only the send queue to a distinct SQE (Send Queue Error) state — the QP stays alive, the receive queue is untouched, and software can recover it directly back to RTS without a full rebuild, because the fault is scoped to exactly the part of the QP that produced it. RC has no SQE state at all. Any completion error on an RC QP takes the *entire* QP to ERROR, both queues, no partial-recovery path.

> [!CAUTION] Gotcha
> Don't assume RC gets the same send-queue-only recovery UC/UD have. It doesn't -- any completion error on an RC QP takes the entire QP to ERROR, both queues, with no SQE-equivalent path to recover just the send side. The reason traces directly back to what RTR→RTS just configured: RC's whole contract is ordered, reliable, acknowledged delivery, backed by a PSN sequence and a retry state machine tracking exactly what's been sent and what's been acknowledged. IBTA-V1 doesn't give RC an SQE-equivalent path the way it does UC/UD, and no source this book found explains why not. A plausible reason, offered as reasoned analysis rather than a documented fact: a fault serious enough to reach a completion puts the QP's sequencing state in doubt, and the send side's own retry logic depends on ACKs arriving correctly over the receive path -- so a fault that looks send-side-only may not actually be isolable from the receive side's sequencing the way UC/UD's simpler, unacknowledged model allows. This book can't confirm hardware couldn't in principle isolate the two; it can only say IBTA-V1 doesn't offer RC that option. What actually happens to a QP's outstanding work once it lands in ERROR — and the isolation guarantee that gives other QPs — is [Single-QP Error Handling](../single-qp-error-handling)'s subject.

**Out of ERROR, there is exactly one door.** No transition takes a QP directly from ERROR back to RTS. The only legal move is ERROR→RESET, after which the QP is rebuilt from scratch through the same INIT→RTR→RTS sequence described above, supplying every required attribute again as if the QP were new. What "cleared" actually means for state that was in flight at the moment of the reset — replay-buffer contents, partially-received messages, DMA already underway — is underdocumented territory this book takes on directly in [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) rather than glossing over it here.

> [!TIP] Why
> Hardware could, in principle, let software assemble a QP's configuration incrementally and only check completeness lazily, the first time the QP actually needs an attribute nobody set. It doesn't, because that would turn a configuration bug into a wire-visible failure — a QP silently sending with a garbage or zero-valued retry timeout, say — discovered only once real traffic depends on it, instead of a rejected state-transition request reported immediately. Enforcing the attribute mask at the transition itself moves every one of those failures to the earliest possible moment, before a single packet carrying that QP's state has gone anywhere. The cost is a rigid, transport-specific mask table baked into the silicon's validation logic rather than a more flexible "fill in whatever, whenever" model — the same trade-off this book's hardware keeps making at every boundary: catch it at the gate, not downstream.

That's the state machine's full input surface, settled once, cleanly, before a later chapter starts asking what happens when one of these fields is wrong at runtime rather than missing at setup. [Single-QP Error Handling](../single-qp-error-handling) is exactly that chapter: what a QP in ERROR actually does to its outstanding work, and the isolation guarantee that keeps one QP's fault from touching any other.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
