---
title: "The Retry and RNR Timeout State Machines"
weight: 5
date: 2026-09-09
publishDate: 2026-09-09
draft: false
description: "A lost packet and a full receive queue are not the same failure, and Reliable Connection doesn't treat them the same way -- two parallel timeout branches, counted independently, both capable of driving a Queue Pair to the same terminal state."
prev: /reliability/replay-buffer-caching
next: /reliability/go-back-n-retransmission
---

## Overview

Two different conditions can leave a requester without an acknowledgment for something it sent: the packet or its acknowledgment was lost somewhere between the two endpoints, or the packet arrived intact but the responder had nowhere to put it and rejected it instead of processing it. Reliable Connection does not fold these into one generic "didn't get acknowledged" state -- they run as two structurally separate state machines, each armed by a different trigger, each counted against its own independent budget, converging only at the very end on the same terminal outcome: a Queue Pair driven to ERROR once its budget for one of them runs out.

## Locator

![The master RNIC block diagram with the Retry Timer block highlighted -- the state machine this chapter covers is logic owned by that block, not a separate structure.](/images/reliability/05-the-retry-and-rnr-timeout-state-machines-locator.svg)

[Retry and Timeout State](/reliability/retry-and-timeout-state) named the live counters this chapter's state machines actually drive. This chapter is the machinery that reads them.

## Block diagram

![Three lanes converging on one QP ERROR node: an ordinary timeout branch (armed on send, expiry triggers go-back-N retransmission, the retry-count ceiling decrements, loops until exhausted), an RNR branch (armed on receiving an RNR NAK, expiry after the advertised backoff delay triggers a same-PSN resend, the RNR-retry ceiling decrements, loops until exhausted), and a third, direct path with no state machine at all -- a Remote Access Error, Remote Operational Error, or Invalid Request NAK reported to the client and driving the QP to ERROR immediately, no retry counter ever consulted.](/images/reliability/05-the-retry-and-rnr-timeout-state-machines-block.svg)

## Deep walkthrough

**The ordinary branch: armed, expired-or-rejected, retried, or disarmed.** A packet leaving the wire arms the timer [Retry and Timeout State](/reliability/retry-and-timeout-state) already introduced, set to fire after the configured ACK timeout duration if nothing acknowledges it first. An acknowledgment arriving before expiry disarms the timer and, if it's the last outstanding request, leaves nothing armed at all; an acknowledgment arriving for a later request while this one is still the oldest outstanding re-arms against whatever is now oldest, per [The Replay Buffer](/reliability/the-replay-buffer)'s cumulative-retirement logic. Two distinct events drain the same counter here, not one: silent expiry with nothing having arrived at all, and an explicit PSN Sequence Error NAK arriving instead of an ACK -- IBTA-V1 groups both under one shared retry counter (a lost packet and a NAK that says "I saw a gap" are the same underlying condition from the requester's side, just discovered two different ways). Either one triggers [Go-Back-N Retransmission](/reliability/go-back-n-retransmission), covered next in full, and decrements the same live retry count. This loop continues for as long as the live retry count stays above zero.

**The RNR branch: a different trigger, a different remedy, a separate budget.** A Receiver-Not-Ready condition isn't silence -- it's an explicit signal, a NAK carrying a backoff delay the responder itself is advertising, sent because the responder lacked a resource the arriving message needed to process it: an empty receive-side buffer for an incoming SEND, most concretely. This is not a lost-packet condition at all: the responder saw the packet, correctly identified that it had nowhere to place it, and said so. The requester's response is correspondingly different from the ordinary branch's go-back-N: rather than resending a range of packets, it waits out the advertised delay -- never shorter than the RNR-NAK timer floor the responder is permitted to advertise -- and then resends the same single packet that triggered the rejection, no more. The RNR-retry count decrements on each such attempt, counted entirely independently of the ordinary retry count next to it.

> [!CAUTION] Gotcha
> The responder's own ceiling on outstanding incoming RDMA Read/Atomic requests ([QP Context](/qp/qp-context)'s responder-side outstanding-read/atomic ceiling) already fully committed when another one arrives is easy to mistake for an RNR trigger -- it isn't one. Per IBTA-V1's own fault-class table, exceeding that ceiling is classified as Invalid Request, not RNR NAK, which means it reaches the *direct* path below (Responder Class C, no retry counter ever consulted) rather than this branch. Only an actual resource shortfall -- most concretely, no receive buffer posted -- gets the RNR treatment.

**Why one shared counter would be the wrong design.** A QP whose peer's receive queue runs empty a few times under a bursty workload, and separately loses a few packets on a lossy path, is facing two unrelated problems. Charging both against one shared budget would let either condition's bad luck exhaust the tolerance for the other -- a QP that happened to hit several RNR conditions early in a connection's life would have less headroom left to tolerate ordinary loss later, for no reason connected to the path's actual loss characteristics. Two independent counters mean a QP's tolerance for one failure mode is never a hidden function of how much of the other failure mode it happened to experience first.

> [!NOTE] Concept
> Two structurally separate state machines, each armed by a different trigger and counted against its own budget, converge on exactly one terminal outcome once *their own* counter is exhausted. Whichever counter reaches zero first -- the ordinary retry-count ceiling or the RNR-retry ceiling -- the QP is driven to the same terminal fault class. Neither of these two branches has its own distinct exit from the other. That's a narrower claim than "this is the only road to that fault class" -- it isn't; see the Gotcha below.

**The one door out, shared by both branches.** Both branches feed the same single exit: per [QP State Machine](/qp/qp-state-machine), the QP is driven to ERROR once its counter runs out. What ERROR actually does to a QP's other outstanding work from that point is [Single-QP Error Handling](/qp/single-qp-error-handling)'s subject, not re-derived here -- this chapter's job stops at naming that exhaustion is the trigger, not at re-explaining what firing that trigger does.

**Most NAK codes never reach this state machine at all.** IBTA-V1 draws a hard line between two families of remotely-detected fault, and this chapter's two branches only cover one of them: "Of the possible NAK codes, two (NAK-Sequence Error and NAK-RNR) indicate operations that should be retried automatically by the requester. The NAK codes other than NAK-Sequence Error and NAK-RNR indicate failures that must be reported to the requester's client immediately and cannot be retried." A Remote Access Error, a Remote Operational Error, or an Invalid Request (an unsupported opcode, a missing FIRST/LAST in a multi-packet message, a length error, or -- per the Gotcha above -- exceeding the negotiated outstanding RDMA Read/Atomic ceiling) never touches either retry counter above -- there's no waiting, no decrementing, no second attempt. The requester reports the failure to its client on the very first occurrence and the QP is driven to error immediately. [Single-QP Error Handling](/qp/single-qp-error-handling) already covers this second family under its own "access-violation error" and "requester-side transport error" names; this chapter's retry/RNR machinery is not the mechanism behind either of them.

> [!CAUTION] Gotcha
> Don't read this chapter's two branches as the complete map of how a QP reaches its terminal fault state from a NAK or a timeout. They're the *retryable* half only. IBTA-V1's own requester fault-behavior classes name a separate, non-retryable half (Remote Access Error, Remote Operational Error, Invalid Request) that reaches the same kind of terminal outcome directly, with no retry counter ever consulted. Both halves can end a QP's normal operation; only one of them is this chapter's subject.

**What happens next.** This chapter named what triggers a retry and what happens when retries run out. It deliberately didn't cover what retransmission actually does to the packets themselves -- [Go-Back-N Retransmission](/reliability/go-back-n-retransmission) is that chapter, immediately next.

> [!TIP] Why
> Treating loss and Receiver-Not-Ready as separate conditions, rather than collapsing both into one generic "retry" state, costs real complexity -- two counters instead of one, two trigger conditions to distinguish instead of one, two different remedies (resend a range versus resend one packet after a specified delay) instead of a single uniform response. That cost buys something back: a receiver that's merely momentarily out of buffer space, a structural hazard rather than a fault, gets a remedy scoped to exactly that condition -- wait and retry the same thing -- without spending down the same budget an actually lossy path would need. Collapsing the two would make an RNR-heavy but otherwise healthy connection look, from the connection's own bookkeeping, indistinguishable from one experiencing real packet loss, right up until it wasn't.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- defines the RNR NAK mechanism, the minimum RNR timer floor, and the independent retry-count semantics this chapter describes. §9.9.2.2 (Requester Side Error Detection -- Remotely Detected Errors) is the direct source for the retryable-vs-immediately-reported split this chapter's Gotcha above draws; Table 56/57's Requester Fault Behavior Classes are the source for which specific NAK codes fall on each side of that split. Table 58 (Responder Error Behavior Summary) is the direct source classifying "too many outstanding RDMA READ or ATOMIC Requests" as Invalid Request rather than RNR NAK -- the correction the ordinary/RNR-branch Gotcha above makes.
