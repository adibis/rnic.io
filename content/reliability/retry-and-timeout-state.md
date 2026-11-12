---
title: "Retry and Timeout State"
weight: 1
date: 2026-09-09
publishDate: 2026-09-09
draft: false
description: "QP Context named four fields and stopped, deferring them to a part of the book that didn't exist yet. This is that part, and these are the fields -- plus the live state built on top of them that no field name alone can carry."
next: /reliability/the-replay-buffer
metaDescription: "The RDMA QP retry fields, ACK timeout, retry count, RNR retry and RNR timer, and the live state an RNIC builds on top of them."
seoTitle: "RDMA Retry and Timeout State"
---

## Overview

Any full-duplex interface that tags its own outstanding requests needs two kinds of bookkeeping on two different sides of the same connection: something on the issuing side tracking what has been sent and not yet confirmed, and something on the receiving side tracking what's expected to arrive next. [QP Context](/qp/qp-context) already covered the receiving side's half of this for ordinary in-sequence traffic -- the receive-side PSN counter, the next sequence number expected. This chapter covers the issuing side's half, and it's a larger set of state than a single counter: not just how many times a request is allowed to be retried before the connection gives up, but which specific outstanding request is currently being timed, whether that timer is armed at all, and how many retry attempts remain against two separately-counted budgets for two structurally different failure conditions.

## Locator

![The master RNIC block diagram with the Retry Timer block highlighted and every other block dimmed -- this chapter covers the live per-QP state that block actually holds.](/images/reliability/01-retry-and-timeout-state-locator.svg)

[QP Context](/qp/qp-context) named the ACK timeout, the retry-count ceiling, the RNR-retry ceiling, and the RNR-NAK timer floor as fields a Reliable Connection Queue Pair carries, then stopped, saying only that they belong to a reliability engine this book hadn't reached yet. This is that engine, and this chapter is where that debt gets paid.

## Block diagram

![Per-QP retry and timeout state: an armed/disarmed timer bit pointing at the oldest unacknowledged PSN, a live retry count counting down from the retry-count ceiling, a live RNR-retry count counting down independently from the RNR-retry ceiling, both feeding toward the QP state machine's ERROR transition.](/images/reliability/01-retry-and-timeout-state-block.svg)

## Deep walkthrough

**Four fields already named, and what was left out of naming them.** [QP Context](/qp/qp-context) listed the ACK timeout, the retry-count ceiling, the RNR-retry ceiling, and the RNR-NAK timer floor among the fields RTR→RTS supplies, alongside the outstanding-read/atomic ceiling pair, without opening any of the four. Two of them are ceilings, not live counts: the retry-count ceiling is the number of retry attempts a single outstanding request is allowed before the connection gives up on it, and the RNR-retry ceiling is the same kind of ceiling for a structurally different condition covered in full in [The Retry and RNR Timeout State Machines](/reliability/the-retry-and-rnr-timeout-state-machines). The ACK timeout and the RNR-NAK timer floor are duration values, not counts -- the ACK timeout sets how long to wait for an acknowledgment before declaring it lost, and the RNR-NAK timer floor sets the floor delay this QP advertises to a peer when it has to reject an arriving message for having nowhere to put it. None of the four is itself the live state a running connection actually consults on every packet; each is a configured limit the live state below is checked against.

**The armed bit, and what it actually points at.** A Reliable Connection Queue Pair's send side keeps a live indicator of whether a retransmission timer is currently running, and if it is, which outstanding request it's timing. This is not one timer per outstanding packet -- a QP with several requests in flight at once doesn't need independent clocks for each of them, because Reliable Connection's acknowledgment model is cumulative: an acknowledgment for a later packet implicitly confirms every earlier one still outstanding at the same time. The timer only ever needs to track the oldest unacknowledged request, because that's the one whose loss would be detected first regardless of how many newer requests are also outstanding and unconfirmed. When an acknowledgment retires the oldest outstanding request, the timer either disarms (nothing left outstanding) or re-arms against whatever is now the new oldest.

**Two live counters, not one, because two conditions are not the same failure.** A request can fail to be acknowledged for two structurally different reasons: the packet or its acknowledgment was lost somewhere in transit, or the packet arrived correctly but the receiver had nowhere to place it and rejected it with a bounded-retry signal instead of processing it. [The Retry and RNR Timeout State Machines](/reliability/the-retry-and-rnr-timeout-state-machines) covers both branches in full; what matters here is that each condition draws down its own live counter, seeded from its own ceiling (the retry-count ceiling for the first, the RNR-retry ceiling for the second), independently.

**What resets a live counter, and what doesn't.** No public source spells out the exact reset semantics in these terms, so what follows is reasoned inference, not a documented fact. The shape that fits the rest of Reliable Connection's contract is that a live retry count is scoped to one outstanding request's own retry history, not the QP's entire lifetime -- a request that eventually succeeds, whether on the first attempt or the last one before exhaustion, doesn't leave a debt behind for the next request to inherit. A QP that has retried one request three times and finally gotten an acknowledgment for it starts the next request's own retry history fresh. This is the only shape consistent with the retry-count ceiling being described as a per-request bound rather than a connection-wide one: a QP that permanently lost a fraction of its retry budget to every past success would eventually be unable to tolerate any loss at all, no matter how healthy the path currently is, which isn't what a bounded-retransmission-attempts contract is for.

**What happens next.** This chapter has named the live state; it hasn't yet named what drives it. [The Replay Buffer](/reliability/the-replay-buffer) is the structure this timer is actually protecting -- the requester's own copy of what it sent, held exactly as long as this chapter's armed timer says it might still be needed.

> [!CAUTION] Gotcha
> A QP's ordinary retry count and its RNR-retry count are separate budgets, counted independently. Burning through several ordinary retries on a lossy path leaves the RNR-retry budget completely untouched, and vice versa -- the two conditions never share a counter, so bad luck in one failure mode never eats into tolerance for the other.

> [!TIP] Why
> Splitting configured ceiling from live, mutable count is the same shape [QP Context](/qp/qp-context) already used for its own outstanding-read/atomic ceilings against the live count of outstanding read/atomic operations tracked against them -- a value software sets once at connection setup, held fixed, checked against a value hardware updates continuously without a software write in the loop. The alternative, deriving remaining retry attempts by comparing a running failure log against the ceiling on every check, would cost a computation on every single timeout instead of a plain decrement, for no benefit: nothing about this QP's behavior needs to know the full history of past failures, only how many attempts remain right now. A single live counter, ticking down from a fixed ceiling and reloadable on success, is the cheapest structure that answers the only question that actually matters at decision time.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- defines the timeout, retry count, RNR retry count, and minimum RNR timer attributes this chapter's live state is built on top of.
