---
title: "Mid-Transaction QP Reset"
weight: 10
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "RESET clears a QP's leftovers -- that much is documented. What counts as a leftover when a multi-packet message is half-arrived, a replay-buffer entry is still waiting on an ACK, and a DMA write is mid-burst is not, and this chapter says so plainly before reasoning through it anyway."
prev: /qp/hardware-assisted-eviction-engines
next: /qp/qp-capacity-and-timing
---

## Overview

RESET is legal from every state a QP can be in, including RTS with operations actively in flight -- and IBTA's own contract for it is thin: outstanding work on both queues gets cleared. What "cleared" means when there's nothing in flight is obvious. What it means when a multi-packet message is half-received, a sent packet is still sitting in a replay buffer waiting on an acknowledgment that will now never come, or a DMA write is mid-burst into host memory, is the one place in this part where documentation runs out and a real implementation has to make choices nothing public writes down.

## Locator

![The master RNIC block diagram with QP Table and QP Cache highlighted -- this chapter's reasoning also reaches into the Replay Buffer, RX Parser/Reassembly, and DMA Engine without owning any of them.](/images/qp/10-mid-transaction-qp-reset-locator.svg)

## Block diagram

![A QP struck by RESET with four satellite structures around it -- Replay Buffer, Rate Limiter, RX Parser/Reassembly, and DMA Engine -- each connected by a dashed line ending in a question mark, posing what happens to each rather than answering it, with the QP's own live PSN counters shown on the node itself.](/images/qp/10-mid-transaction-qp-reset-block.svg)

## Deep walkthrough

**What actually exists to reconcile.** A QP mid-transaction is never just "a state enum plus two PSN counters" -- by the time RESET can be issued against it, RTS has let it accumulate real state elsewhere in the adapter, all of it this book has already named in earlier chapters: unacknowledged sent packets held in the reliability engine's replay buffer, waiting on an ACK that RESET makes moot (Part 5 (Reliability Engine)); a partially-arrived multi-packet message sitting in RX reassembly state, if the last thing this QP received was a FIRST or MIDDLE packet rather than a LAST (Part 4 (RX Pipeline)); a DMA write that may be actively mid-burst into host memory this instant, not yet committed in full (Part 8 (DMA Engine)); the live receive-side and send-side PSN counters this book's QP Context chapter (1.1) already covered; and this QP's live congestion/rate-limiter state -- current injection rate, outstanding DCQCN feedback timers -- which Part 6 (Congestion Control Engine) owns and which is built the same table-plus-working-set-plus-eviction way the Replay Buffer is. RESET's job is to leave none of that in a state that corrupts this same QP once it's rearmed through INIT→RTR→RTS again.

> [!CAUTION] Gotcha
> Don't conflate this QP rearming itself (ERROR→RESET, same QP object) with [The QP Table](../the-qp-table)'s ABA-shaped concern about a *different*, unrelated QP later reusing this same table entry after this QP is destroyed outright. Same table entry, two different hazards.

But the specific sequencing that gets there -- what gets aborted, what gets allowed to finish, what gets silently dropped -- is not something any public source for a real RNIC documents, for any of these five, rate-limiter state included.

**Where this is checked, and why that's an honest non-answer here.** For every other chapter in this part, "where checked" points at a real pipeline stage: a table lookup, a comparison against a live counter, a validation gate. For mid-transaction reset, the honest answer is that this book cannot point at a specific stage and say "this is where the teardown sequencing happens," because no vendor has published that sequencing. What can be said with confidence is *what has to be true* when RESET completes: the QP must not still be consuming replay-buffer capacity for packets nobody will ever retransmit again, RX reassembly state for an abandoned message must not leak into this same QP's next incarnation once it's rearmed, and a DMA write must not be left in a state where its completion status is ambiguous once RESET has completed.

**When it fails.** There is no "failure mode" to report here in the sense the rest of this part uses that phrase -- RESET is defined to succeed unconditionally from any state. The risk this chapter is actually naming is a correctness risk, not a rejected transition: if a mid-burst DMA write is allowed to keep writing after the QP that issued it has already moved to RESET and been reassigned, or if replay-buffer capacity isn't actually reclaimed until some later, unspecified point, that's a bug class this book can point at without being able to say how any specific implementation avoids it.

**What happens next.** Once RESET completes and the QP re-enters INIT, it starts the same hardware-enforced sequence [QP State Machine](../qp-state-machine) already covered, as if built fresh -- nothing about mid-transaction reset changes that forward path. The open question this chapter leaves is entirely about what happened to the state that used to belong to this QP's *previous* incarnation on the way there.

> [!TIP] Why
> Three real design questions sit inside "clear the leftovers," and a real implementation has to answer all three, even though none of them is settled in anything this book found public. First: does a DMA write already in progress get allowed to run to completion, or aborted mid-burst? Letting it finish is simpler and avoids a torn, half-written destination buffer, but only if nothing downstream is watching for the operation's completion status anymore -- and something might be, if completion reporting (Part 7 (Completion Queue / Event Management)) hasn't itself been told the QP is gone. Aborting mid-burst avoids that ambiguity but risks leaving the destination in a state no future reader of that buffer was told to expect, which is arguably worse if anything assumed atomicity at the message boundary. Second: does the replay buffer's unacked data get force-freed the instant RESET is issued, or held until its retry timer would naturally have expired? Force-freeing reclaims capacity immediately, which matters under the same cache/working-set pressure [Part 1](/qp/) (Queue Pair Management)'s caching and eviction chapters (1.3, 1.8) already established is real; holding it costs nothing except capacity for something that's provably never going to be used again, since RESET has already decided this QP's sequencing state is void. Third: what happens if RESET is issued in the exact window where a retry retransmission has already left the TX pipeline for this QP but hasn't yet reached the wire -- does that packet still go out, referencing a QP context that's already been reset underneath it? Each of these is a real race a real implementation has had to resolve one way or another; this book states the questions with confidence and declines to guess at any vendor's specific answer.

> [!IMPORTANT] Food for Thought
> Pick one of the three races above -- DMA-write-in-progress, replay-buffer force-free timing, or a retransmission already in flight when RESET lands -- and argue both sides. What would you actually build, and what would you give up either way?

This is not a new class of problem. chipdv.io's [Mid-Sim Reset](https://chipdv.io/reuse/mid-sim-reset-plumbing/) covers the same shape of hazard -- a resource torn down while something is still actively using it -- from a testbench-verification angle: a reset arriving mid-simulation while a driver, monitor, or sequence still holds state that assumes the DUT hasn't moved. The domain is different, but the question underneath it isn't: what does "reset" actually have to reconcile when it doesn't arrive at a quiet moment.

This is the last chapter in this part built around one QP at a time. [QP Capacity and Timing](/qp/qp-capacity-and-timing), next, closes Part 1 by asking how many of these can exist together, and what it costs in silicon to let them.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
- No IETF or IBTA standard governs the exact hardware sequencing of a mid-transaction reset (as opposed to the RESET state transition itself, which IBTA-V1 does define) -- it is implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above, not a standards document.
