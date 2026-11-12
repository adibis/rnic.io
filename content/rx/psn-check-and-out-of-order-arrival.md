---
title: "PSN Check and Out-of-Order Arrival"
weight: 3
date: 2026-09-09
publishDate: 2026-09-09
draft: false
description: "A packet sequence number is a transaction tag, not a timestamp -- and the three-way branch it drives (matches, too low, too high) is the single gate every parsed packet has to clear before it means anything at all. This chapter is that gate, and the one place a resolved packet actually hands off to memory."
prev: /rx/rx-packet-parse
next: /rx/multi-packet-reassembly
metaDescription: "The RoCEv2 PSN check: a packet's sequence number is expected, a duplicate, or reveals a gap, and that three-way branch gates everything after it."
seoTitle: "RDMA PSN Check and Out-of-Order Arrival"
---

## Overview

A packet sequence number exists to solve one problem: let a hardware receiver, with no notion of wall-clock time and no reliable way to know what a sender has already retransmitted, decide with certainty whether an arriving packet is the one it expects, one it has already seen, or one that implies something was lost. This is a transaction-tag problem, not a timing problem. A Queue Pair's receiving side holds exactly one number that matters for this decision: the sequence number it expects next.

> [!WARNING] Analogy
> This is the same shape of question a tagged, non-blocking interface answers for any response that can arrive out of order relative to when it was issued -- match, and it's the one you were waiting for; too early, a duplicate; too late, something's missing. Everything this chapter covers is what happens when a parsed, validated packet's own sequence number is compared against it.

## Locator

![The master RNIC block diagram with the RX PSN Check stage highlighted, positioned immediately after RX Parser / Reassembly and with an arrow reaching into the DMA Engine block for the handoff this chapter closes with.](/images/rx/03-psn-check-and-out-of-order-arrival-locator.svg)

[RX Packet Parse](/rx/rx-packet-parse) confirmed a packet's framing, identified its destination Queue Pair, and validated its integrity -- but deliberately stopped short of deciding whether the packet is actually allowed to proceed. This chapter is that decision.

## Block diagram

![A three-way branch fed by one comparison -- arriving PSN against the destination QP's next-expected PSN -- with the "matches" path continuing straight through to permission checking and DMA handoff, the "lower than expected" path looping back to a re-acknowledgment box that never touches the Receive Queue, and the "higher than expected" path terminating in a reject-and-signal box captioned with the PSN the sender should resume from.](/images/rx/03-psn-check-and-out-of-order-arrival-block.svg)

{{< flow "go-back-n" >}}

{{< flow "ack-timeout" >}}

## Deep walkthrough

**The comparison, and the three outcomes it produces.** A Reliable Connection Queue Pair tracks the next sequence number it expects to receive as part of its live context -- the same field this book has already named without opening. Every packet that survives parsing gets its own sequence number compared against that expectation, and the comparison has exactly three possible outcomes. A match means the packet is exactly the one expected, and processing continues. A sequence number lower than expected means this packet is a duplicate -- almost certainly a retransmission of something already received, most likely because the sender's earlier acknowledgment never arrived back. A sequence number higher than expected means a gap: something between what was last received and what just arrived is missing, whether lost in transit or never sent yet in the sender's own ordering.

**A duplicate does not get reprocessed, and does not touch the Receive Queue -- for the families where a bare re-acknowledgment is actually a complete response.** The correct response to a lower-than-expected sequence number is not to run the packet through the rest of the pipeline a second time -- its effects, whatever they were, already happened once. Re-executing a SEND's Receive Queue consumption a second time would silently corrupt the ordering [The Receive Queue Descriptor and Ring](/rx/the-receive-queue-descriptor-and-ring) established as this structure's entire reason for existing; re-executing a memory placement a second time risks writing stale data over whatever has landed there since. For SEND and WRITE, the architecturally correct response is narrower: acknowledge the sequence number again, so the sender's own retry logic sees confirmation and stops retransmitting, without repeating any of the work the first, correctly-ordered arrival already did.

That narrower response doesn't cover every opcode family, though. [The Operation Catalog](/overview/operation-catalog) already established that RDMA READ has no separate bare-acknowledgment packet at all -- its response payload doubles as its own ACK -- so a duplicate `RDMA_READ_REQUEST` can't be resolved by "acknowledging the sequence number again"; there is no such packet for READ to send. What a real responder does instead -- regenerate and resend the actual read-response data, or something else -- is not something this book resolves here. The same gap applies to Atomics: `ATOMIC_ACKNOWLEDGE` carries the pre-operation value the requester still needs, which a bare re-ack can't supply. Both are named as an open exception to this chapter's general rule, not folded silently into it.

**A gap is not buffered -- it is rejected, with enough information to resume.** This is the point at which Reliable Connection's receiving side makes a real architectural choice, not an accident of implementation: a packet arriving out of order is not held in some per-Queue-Pair reorder structure waiting for the missing one to show up later. It is rejected outright, with a signal back to the sender naming the sequence number the receiver is actually waiting for, so the sender knows exactly where to resume rather than guessing. This is a deliberate rejection of the alternative -- buffering out-of-order arrivals and resequencing them once the gap fills -- and the alternative is not free: it would require a per-Queue-Pair reorder buffer sized to however far ahead a sender might legitimately race, real on-chip storage spent on a condition every well-behaved connection experiences rarely. What actually happens to the sender's own outstanding, unacknowledged work once this rejection reaches it belongs to the not-yet-written reliability engine, not this chapter -- this chapter's job stops at naming that the receiver rejects rather than buffers, and why that is a load-bearing design choice rather than an oversight.

**Once a packet matches, one more gate stands before memory, and this chapter names it without re-deriving it.** A packet whose sequence number matches is not yet cleared to touch host memory. Whatever address or key it carries -- a remote address and its authorizing key for a one-sided operation, or nothing at all for a two-sided one consuming a Receive Queue entry instead -- still has to resolve against the Key Table and pass whatever permission and bounds checking that resolution requires. That check, and everything it can reject, belongs entirely to [Permission and Bounds Violations](/mr/permission-and-bounds-violations); this chapter does not repeat it.

**The handoff: what has to be true before a packet reaches memory, not how it gets placed.** A packet that clears the sequence-number gate and clears permission and bounds checking is, at that instant, ready to have its payload placed in host memory. The mechanics of that placement -- resolving a virtual address, walking a scatter-gather descriptor chain, the ordering guarantees a real implementation provides -- belong entirely to the not-yet-written DMA Engine. This chapter's job ends at naming the handoff itself: a resolved, authorized operation, carrying a validated address and length, crossing from this part's territory into that one's. Nothing about how the DMA engine actually moves those bytes is this chapter's concern.

**What happens next, for the one case this chapter has deliberately deferred.** Everything above describes a single packet's fate. A message spanning more than one packet -- opened by a FIRST, continued by however many MIDDLE packets, closed by a LAST -- needs state that persists across every packet in that sequence, not just a single comparison against the next one. [Multi-Packet Reassembly](/rx/multi-packet-reassembly) is that state, immediately next.

> [!TIP] Why
> Rejecting an out-of-order arrival instead of buffering it trades a recurring cost for a one-time savings: every gap forces a full go-back-N retransmission of everything from the missing packet forward, even packets that technically already arrived correctly and are about to be resent and discarded a second time -- wasted link bandwidth on every gap, not a theoretical inefficiency. What it avoids is a per-Queue-Pair reorder buffer sized to accommodate however far out of order a sender might legitimately get, real on-chip storage that scales with the number of concurrently active Queue Pairs the same way every other per-QP structure in this book does, spent on a condition well-behaved traffic rarely triggers. A design that instead built the reorder buffer would pay silicon continuously, for every Queue Pair, to save bandwidth only on the rare packet that actually arrives out of order -- reject-and-resume spends bandwidth rarely to avoid spending silicon always. Neither choice is free; Reliable Connection's architecture has made its choice, and this chapter names which one and why it is defensible, not merely convenient.

[Multi-Packet Reassembly](/rx/multi-packet-reassembly) is next: the state a message spanning more than one packet needs, and this chapter's single-packet gate does not provide.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- defines the packet sequence number field this chapter's comparison operates on and the duplicate/gap semantics this chapter describes.
