---
title: "Mid-Transaction Reliability Reset"
weight: 9
date: 2026-09-09
publishDate: 2026-09-09
draft: false
description: "Mid-Transaction QP Reset named replay-buffer contents and armed timers among the leftovers RESET has to reconcile, and left both unresolved. This chapter resolves them, and finds one hazard neither chapter can fully close."
prev: /reliability/replay-buffer-capacity-and-backpressure
---

## Overview

RESET is legal from any state a Reliable Connection Queue Pair can be in, including RTS with a timer currently armed and Replay Buffer entries currently held against outstanding, unacknowledged requests. IBTA's own contract for RESET is thin on specifics: outstanding work gets cleared. What clearing means for this part's own state -- a timer mid-countdown, entries waiting on acknowledgments that may now never arrive -- is where this chapter has to reason past what's written down, the same territory [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) already opened for the QP's other in-flight state.

## Locator

![The master RNIC block diagram with the Retry Timer and Replay Buffer both highlighted -- this chapter's reasoning also reaches into the QP Table and QP Cache without owning either.](/images/reliability/08-mid-transaction-reliability-reset-locator.svg)

[Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) named this part's two resources -- an armed timer and outstanding Replay Buffer entries -- among the state a RESET has to reconcile, and left both open, deferring them to a part of the book that didn't exist yet. This chapter is that part's answer.

## Block diagram

![A QP with an armed timer and three outstanding Replay Buffer entries at the moment RESET lands: the timer disarming immediately, the three entries being freed rather than drained, and a dashed, unresolved arrow captioned "late-arriving ACK for a pre-reset PSN" pointing at the now-cleared QP context with a question mark.](/images/reliability/08-mid-transaction-reliability-reset-block.svg)

## Deep walkthrough

**The timer: disarming is the only choice that makes sense.** [Retry and Timeout State](/reliability/retry-and-timeout-state)'s armed timer exists to trigger a retransmission if an acknowledgment doesn't arrive. Once RESET lands, there is nothing left to retransmit toward in any sense the timer's original arming intended -- the QP's outstanding requests are being cleared, not merely paused, and letting the timer fire afterward and attempt a retransmission against a QP that's mid-reset (or already rebuilt into a new connection) would be actively wrong, not just unnecessary. Disarming immediately, as part of RESET's own handling, is the only choice consistent with everything else this book has established about what RESET does to outstanding state.

**The Replay Buffer: freed, not drained.** [Eviction Policy](/qp/eviction-policy)'s ordinary eviction path writes back a dirty entry before freeing its slot, because that entry's data still matters to something and nowhere else holds a copy of it. A Replay Buffer entry is different for a reason [The Replay Buffer](/reliability/the-replay-buffer) already established: it was never the only copy of anything.

> [!CAUTION] Gotcha
> It's tempting to assume a Replay Buffer entry needs the same dirty-writeback handling an evicted cache entry gets. It doesn't. What it holds is a reference back to the original Work Queue Element's source buffer, not a duplicate of the payload -- the bytes a drain would supposedly be protecting are still sitting exactly where they always were, in host memory, untouched by RESET. Freeing these entries outright, with no write-back step, is the correct and only sensible handling.

**The hazard neither chapter fully closes: a late acknowledgment for a PSN that no longer means anything.** [The QP Table](/qp/the-qp-table) already named the general shape of this problem for a QP number reused after destruction, and [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) named it again for a QP rearming itself through ERROR→RESET rather than being destroyed outright. This part has its own version: an acknowledgment for a pre-reset PSN, delayed somewhere in the network, arriving after RESET has already cleared this QP's sequence-number state and the QP has since been reconfigured -- potentially with PSN numbering that legitimately overlaps what the stale acknowledgment is still referencing. Whether a real implementation quarantines a PSN space for some interval after RESET, or relies entirely on other checks to reject a stale arrival that happens to land during the gap, is not documented anywhere this book found. This chapter doesn't resolve it any more than the two chapters before it did -- it names the reliability-specific instance of the same open hazard, in the same spirit those chapters left it, rather than pretending a clean answer exists where none is on record.

> [!IMPORTANT] Food for Thought
> A stale acknowledgment for a pre-reset PSN arrives after the QP has been reconfigured with numbering that happens to overlap it. If you had to design the fix, would you quarantine the PSN space for some interval after RESET, or lean entirely on other checks to reject the stale arrival if it lands during the gap? What would each choice cost?

**What this part's contribution to RESET actually amounts to.** [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) listed five satellite structures RESET has to reconcile without resolving any of them in that chapter. This part resolves two: an armed timer, which disarms cleanly with no ambiguity, and Replay Buffer contents, which free cleanly with no ambiguity. The PSN-reuse hazard above is not this part's to resolve alone -- it's the same open question [The QP Table](/qp/the-qp-table) already carries, now confirmed to reach into this part's own state as well rather than being contained to QP identity alone.

> [!TIP] Why
> Freeing Replay Buffer entries outright rather than attempting any kind of graceful drain is the cheap, correct answer specifically because there's no safe alternative it would be trading against -- unlike an evicted cache entry, which has a real reason to be written back before its slot is reused, a reset QP's replay state has no destination worth preserving it for. Building a drain path anyway, on the theory that some future acknowledgment might still arrive and ought to be honored, would mean holding state for a connection that no longer exists in any form software can act on, indefinitely, against acknowledgments that arrive rarely if ever. The cost of getting this wrong in the other direction -- a stale acknowledgment resolving against reused state after RESET -- is an open hazard this chapter names rather than solves, but building elaborate protection against it inside the Replay Buffer specifically, rather than at the PSN-reuse boundary [The QP Table](/qp/the-qp-table) already identified as the actual source of the ambiguity, would be solving the problem in the wrong place.

This closes [Reliability Engine](/reliability/). [Congestion Control Engine](/congestion/) is next.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- defines RESET's legality from any state and its general contract for outstanding work.
