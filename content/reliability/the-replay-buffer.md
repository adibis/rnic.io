---
title: "The Replay Buffer"
weight: 2
date: 2026-09-09
publishDate: 2026-09-09
draft: true
description: "A reliable sender can't forget what it sent the instant it sends it -- only once something confirms the other side actually has it. This chapter is where that in-between state actually lives, and what it costs to hold it."
prev: /reliability/retry-and-timeout-state
next: /reliability/replay-buffer-caching
---

## Overview

A sender operating over an unreliable channel that still promises reliable delivery faces one unavoidable requirement: it cannot discard what it sent until something confirms the other side received it. An ordinary transmit path is free to release a buffer's contents the instant the bytes leave the wire -- once sent, a wire's own job is done. A Reliable Connection cannot make that assumption, because "sent" and "delivered" are not the same event, and the gap between them is exactly where loss happens. Something has to hold a retrievable record of every unacknowledged operation for as long as [Retry and Timeout State](/reliability/retry-and-timeout-state)'s timer says it might still need to be resent. That something is the Replay Buffer.

## Locator

![The master RNIC block diagram with the Replay Buffer highlighted -- the same visual family as the QP Table/Cache and MR Table/Key Cache pairs, and every other block dimmed.](/images/reliability/02-the-replay-buffer-locator.svg)

[Retry and Timeout State](/reliability/retry-and-timeout-state) named the live counters and the armed timer bit tracking whether a retransmission might be needed. This chapter covers what actually gets retransmitted when one is.

## Block diagram

![The Replay Buffer as entries keyed by QP and PSN, each holding the header fields needed to rebuild a packet plus a reference back to the original WQE's source buffer rather than a duplicate payload copy, with an ACK arriving to retire one entry and a timer expiry reading one back out for retransmission.](/images/reliability/02-the-replay-buffer-block.svg)

## Deep walkthrough

**What this structure is addressed by.** A Replay Buffer entry is found by the same two coordinates every per-packet structure in this book has used so far: which Queue Pair, and which packet sequence number within that Queue Pair's own send-side numbering. An acknowledgment arriving names a PSN; a timer expiring names whichever PSN [Retry and Timeout State](/reliability/retry-and-timeout-state) was tracking as the oldest outstanding one. Both operations resolve to the same addressing scheme, just triggered from opposite directions -- one retiring an entry, the other reading one back out.

**What actually has to be held, and what almost certainly doesn't.** The naive answer -- keep a full byte-for-byte copy of every unacknowledged packet's payload -- is not the only shape that satisfies the requirement, and probably not the cheapest one. [TX Packet Build](/tx/tx-packet-build) already established that an outgoing packet's payload is read out of host memory via the sending Work Queue Element's own scatter-gather list, resolved against that memory's registration the same way any send resolves it. Nothing about a retransmission requires that data to have moved anywhere else in the meantime -- the original source buffer is still registered, still valid, and (barring a software error this book isn't responsible for reasoning about) still holds the same bytes it held the first time. This book states the shape below as reasoned architectural inference rather than a documented fact -- no public source confirms which shape a specific shipped implementation actually uses.

> [!CAUTION] Gotcha
> A Replay Buffer entry is easy to picture as a second copy of the packet's payload. The lighter, more plausible design holds only what's needed to rebuild the packet deterministically: the header fields a retransmission has to reproduce exactly (opcode, destination information, the PSN being resent), plus a reference back to the original WQE's scatter-gather list -- not a second copy of everything it points to. A reference-based design avoids paying for the same bytes twice, once in host memory and again in a dedicated replay structure.

**What bounds how large this structure needs to be.** A Replay Buffer is not an open-ended structure sized against every packet a QP has ever sent -- only against packets currently outstanding and unacknowledged, a count that's naturally bounded by how deep a Queue Pair's own Send Queue can run ahead of its acknowledgments. A connection with a healthy, low-latency path to its peer keeps very little outstanding at once, because acknowledgments return almost as fast as requests go out; a connection with a long round-trip time or a peer that batches its acknowledgments can have substantially more outstanding simultaneously. This is not a fixed number this book can state as an architectural constant -- it's a function of round-trip time, acknowledgment behavior, and however many requests a given workload keeps in flight at once, and [Replay Buffer Capacity and Backpressure](/reliability/replay-buffer-capacity-and-backpressure) covers directly what happens when a real implementation's provisioned capacity is smaller than what a workload would otherwise want to keep outstanding.

**What retiring an entry actually requires.** An acknowledgment doesn't just need to name a PSN -- because Reliable Connection's acknowledgments are cumulative, one acknowledgment for a given PSN confirms every earlier outstanding PSN at the same time, not just the one it names. Retiring a Replay Buffer entry on acknowledgment is therefore not always a single-entry operation; it can be a range operation, freeing every entry from the previous confirmed point up through the newly acknowledged one in a single pass. [Retry and Timeout State](/reliability/retry-and-timeout-state)'s timer re-arming logic depends on this directly -- disarming and re-arming against the new oldest outstanding entry only makes sense once every entry the acknowledgment actually covers has already been retired.

**What happens next.** Everything above assumes this structure's contents are resident somewhere accessible when they're needed. Whether that's on-chip SRAM, a backing store, or some working-set split between the two -- and what a real deployment's traffic pattern does to that split -- is [Replay Buffer Caching](/reliability/replay-buffer-caching)'s subject next.

> [!TIP] Why
> Holding a reference back to the original source buffer instead of a duplicate payload copy is the cheaper design specifically because RDMA's own memory-registration model already guarantees the source data stays put and stays valid for as long as the application hasn't explicitly deregistered it -- a guarantee a conventional network stack, which typically hands ownership of a buffer to the kernel and lets the application reuse it immediately, doesn't get to assume. The cost of that cheaper design is a hard dependency on the source buffer's continued validity: an application that deregisters or reuses a buffer while a retransmission might still need to read from it creates exactly the kind of hazard [Mid-Transaction Deregistration](/mr/mid-transaction-deregistration) already covers for the read side of the same problem. A full-copy design would avoid that dependency entirely, at the cost of holding every in-flight byte twice for as long as it remains unacknowledged -- real storage spent to buy independence from an assumption RDMA's own registration model already exists to guarantee.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- defines the cumulative acknowledgment semantics this chapter's retirement logic depends on.
- No IETF or IBTA standard governs the Replay Buffer's own storage shape -- what it actually holds and how -- since that's implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment of that question is reasoned architectural inference, stated as such, not a citation.
