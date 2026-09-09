---
title: "The Retry and RNR Timeout State Machines"
weight: 4
date: 2026-09-09
publishDate: 2026-09-09
draft: true
description: "A lost packet and a full receive queue are not the same failure, and Reliable Connection doesn't treat them the same way -- two parallel timeout branches, counted independently, both capable of driving a Queue Pair to the same terminal state."
prev: /reliability/replay-buffer-caching
next: /reliability/go-back-n-retransmission
---

## Locator

![The master RNIC block diagram with the Retry Timer block highlighted -- the state machine this chapter covers is logic owned by that block, not a separate structure.](/images/reliability/04-the-retry-and-rnr-timeout-state-machines-locator.svg)

[Retry and Timeout State](/reliability/retry-and-timeout-state) named the live counters this chapter's state machines actually drive. This chapter is the machinery that reads them.

## Overview

Two different conditions can leave a requester without an acknowledgment for something it sent: the packet or its acknowledgment was lost somewhere between the two endpoints, or the packet arrived intact but the responder had nowhere to put it and rejected it instead of processing it. Reliable Connection does not fold these into one generic "didn't get acknowledged" state -- they run as two structurally separate state machines, each armed by a different trigger, each counted against its own independent budget, converging only at the very end on the same terminal outcome: a Queue Pair driven to ERROR once its budget for one of them runs out.

## Block diagram

![Two parallel state-machine lanes -- an ordinary timeout branch (armed on send, expiry triggers go-back-N retransmission, retry_cnt decrements, loops until exhausted) and an RNR branch (armed on receiving an RNR NAK, expiry after the advertised backoff delay triggers a same-PSN resend, rnr_retry decrements, loops until exhausted) -- both converging on a single QP ERROR node.](/images/reliability/04-the-retry-and-rnr-timeout-state-machines-block.svg)

## Deep walkthrough

**The ordinary branch: armed, expired, retried, or disarmed.** A packet leaving the wire arms the timer [Retry and Timeout State](/reliability/retry-and-timeout-state) already introduced, set to fire after the configured `timeout` duration if nothing acknowledges it first. An acknowledgment arriving before expiry disarms the timer and, if it's the last outstanding request, leaves nothing armed at all; an acknowledgment arriving for a later request while this one is still the oldest outstanding re-arms against whatever is now oldest, per [The Replay Buffer](/reliability/the-replay-buffer)'s cumulative-retirement logic. Expiry with nothing having arrived -- no acknowledgment, and no explicit rejection either -- decrements the live retry count and triggers [Go-Back-N Retransmission](/reliability/go-back-n-retransmission), covered next in full. This loop continues for as long as the live retry count stays above zero.

**The RNR branch: a different trigger, a different remedy, a separate budget.** A Receiver-Not-Ready condition isn't silence -- it's an explicit signal, a NAK carrying a backoff delay the responder itself is advertising, sent because that responder's receive-side buffer was empty when a message needing one arrived. This is not a lost-packet condition at all: the responder saw the packet, correctly identified that it had nowhere to place it, and said so. The requester's response is correspondingly different from the ordinary branch's go-back-N: rather than resending a range of packets, it waits out the advertised delay -- never shorter than `min_rnr_timer`, the floor the responder is permitted to advertise -- and then resends the same single packet that triggered the rejection, no more. The RNR-retry count decrements on each such attempt, counted entirely independently of the ordinary retry count next to it.

**Why one shared counter would be the wrong design.** A QP whose peer's receive queue runs empty a few times under a bursty workload, and separately experiences a few genuinely lost packets on a lossy path, is facing two unrelated problems. Charging both against one shared budget would let either condition's bad luck exhaust the tolerance for the other -- a QP that happened to hit several RNR conditions early in a connection's life would have less headroom left to tolerate ordinary loss later, for no reason connected to the path's actual loss characteristics. Two independent counters mean a QP's tolerance for one failure mode is never a hidden function of how much of the other failure mode it happened to experience first.

> [!NOTE] Concept
> Two structurally separate state machines, each armed by a different trigger and counted against its own budget, converge on exactly one terminal outcome. Whichever counter reaches zero first -- ordinary `retry_cnt` or RNR's `rnr_retry` -- the QP is driven to the same ERROR state. Neither branch has its own distinct exit.

**The one door out, shared by both branches.** Whichever counter reaches zero first -- ordinary `retry_cnt` or RNR's `rnr_retry` -- the outcome is identical: the outstanding request completes in error, and per [QP State Machine](/qp/qp-state-machine), the QP is driven to ERROR. Neither branch has its own distinct terminal state; both feed the same single exit. What ERROR actually does to a QP's other outstanding work from that point forward is [Single-QP Error Handling](/qp/single-qp-error-handling)'s subject, not re-derived here -- this chapter's job stops at naming that exhaustion is the trigger, not at re-explaining what firing that trigger does.

**What happens next.** This chapter named what triggers a retry and what happens when retries run out. It deliberately didn't cover what retransmission actually does to the packets themselves -- [Go-Back-N Retransmission](/reliability/go-back-n-retransmission) is that chapter, immediately next.

> [!TIP] Why
> Treating loss and Receiver-Not-Ready as genuinely separate conditions, rather than collapsing both into one generic "retry" state, costs real complexity -- two counters instead of one, two trigger conditions to distinguish instead of one, two different remedies (resend a range versus resend one packet after a specified delay) instead of a single uniform response. That cost buys a real property: a receiver that's merely momentarily out of buffer space, a structural hazard rather than a fault, gets a remedy scoped to exactly that condition -- wait and retry the same thing -- without spending down the same budget a genuinely lossy path would need. Collapsing the two would make an RNR-heavy but otherwise perfectly healthy connection look, from the connection's own bookkeeping, indistinguishable from one experiencing real packet loss, right up until it wasn't.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- defines the RNR NAK mechanism, the minimum RNR timer floor, and the independent retry-count semantics this chapter describes.
