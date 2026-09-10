---
title: "Multi-Packet Reassembly"
weight: 4
date: 2026-09-09
publishDate: 2026-09-09
draft: true
description: "RETH names a destination address exactly once, on a multi-packet RDMA WRITE's first packet, and never again. Every packet after it has to land somewhere that first packet never restated -- which means something has to remember, and this chapter is what that something actually holds for the RETH-seeded case specifically."
prev: /rx/psn-check-and-out-of-order-arrival
next: /rx/single-qp-rx-stalls
---

## Overview

A FIRST packet opens a message; however many MIDDLE packets continue it; a LAST packet closes it. Between FIRST and LAST, every packet belonging to that message has to land in the correct place relative to the ones before it. This chapter covers the RDMA WRITE case specifically: [The Operation Catalog](/overview/operation-catalog) already established that RETH, the extended header carrying a remote address and length, appears only on WRITE's FIRST and ONLY packets, never on MIDDLE or LAST. Something has to hold that address, and how far into the message the last packet advanced it, for as long as the message stays open. This chapter is that something -- for the family whose addressing is RETH-seeded. Two of this book's other multi-packet-capable families are seeded differently, and are named below rather than silently folded into this chapter's RETH-specific model.

## Locator

![The master RNIC block diagram with RX Parser / Reassembly highlighted, the same block Chapter 4.2 targets, this time zoomed to the state that persists across packets rather than the parsing logic that runs once per packet.](/images/rx/04-multi-packet-reassembly-locator.svg)

[PSN Check and Out-of-Order Arrival](/rx/psn-check-and-out-of-order-arrival) covered a single packet's admission gate. This chapter covers what a *message* needs -- state that outlives any one packet, because a message can span several of them.

## Block diagram

![One open reassembly context per receiving Queue Pair, for a multi-packet RDMA WRITE: a destination address seeded from RETH on the FIRST packet, a running byte offset advancing by each MIDDLE and LAST packet's payload length, and a remaining-length counter reaching zero exactly when LAST arrives -- shown alongside a second Queue Pair's identical, entirely independent context, illustrating that this state is bounded per-QP rather than a shared working set.](/images/rx/04-multi-packet-reassembly-block.svg)

## Deep walkthrough

![A four-packet RDMA WRITE message (FIRST, MIDDLE, MIDDLE, LAST) with the reassembly context's two live fields shown updating at each step: FIRST seeds the offset and remaining-length counter from RETH, each MIDDLE advances the offset and shrinks the counter by its own payload length, and LAST's arrival is exactly the moment the counter reaches zero and the context closes.](/images/rx/04-multi-packet-reassembly-sequence.svg)

**What actually has to be remembered, and why it is so little.** A FIRST packet's RETH names a starting address, a total length, and the key authorizing the write. Every packet after it carries none of that -- only payload. What this stage has to hold, for the life of one open message, is exactly enough to place the next packet's payload correctly without RETH repeating itself: the address the next byte belongs at, and how many bytes remain before the message is complete. Both update by the same amount on every MIDDLE and LAST packet -- the address advances by that packet's payload length, and the remaining count shrinks by the same amount, reaching zero exactly when LAST arrives and closes the message out. This is a small amount of state, not because the problem is unimportant, but because RETH already did the expensive part once, on FIRST, and every packet after it only has to advance a position rather than resolve one from scratch.

**Two other multi-packet families, seeded differently, not this chapter's subject.** RDMA WRITE isn't the only opcode family with FIRST/MIDDLE/LAST framing. A multi-packet SEND (`SEND_FIRST`/`SEND_MIDDLE`/`SEND_LAST`) carries no RETH at all -- [The Operation Catalog](/overview/operation-catalog) shows it as BTH-only on every packet in the sequence. Its destination address comes from the Receive Queue entry popped at FIRST instead: [The Receive Queue Descriptor and Ring](/rx/the-receive-queue-descriptor-and-ring) already establishes that a posted RQ entry names a destination buffer directly, no RETH involved. A multi-packet `RDMA_READ_RESPONSE` is stranger still: it carries no RETH either (BTH-only on FIRST/MIDDLE, per the same catalog), and it isn't received by the same side this chapter has been describing -- it lands at the original requester, whose destination address for the data coming back was named in the local WQE that requester posted for its own `RDMA_READ_REQUEST`, not in anything arriving on the wire. Both are real reassembly problems with the same shape as this chapter's -- something has to hold a running address and remaining length across MIDDLE packets -- seeded from a different source than RETH. Neither is worked through here; naming them is this chapter's only obligation to the reader who might otherwise assume RETH-seeding is universal.

**Why this is not the same shape of structure as the QP Cache or the Key Cache.** [QP Caching](/qp/qp-caching) and [Key Caching](/mr/key-caching) both hold a working set that a demand-driven lookup can hit or miss -- an arbitrary QP number or key arrives, and the cache either already holds the entry or has to fetch it. This chapter's state has no lookup and no miss path: a receiving Queue Pair either has a message currently open, in which case an arriving MIDDLE or LAST packet updates that one context directly, or it does not, in which case only a FIRST or ONLY packet is legal. There is nothing to search and nothing that can miss -- the state belongs to exactly one Queue Pair at a time, addressed by construction rather than by lookup, which is a different shape from every caching chapter this book has covered so far.

**Bounded by the protocol itself, not by an implementation choice.** A Reliable Connection Queue Pair's packets arrive in strict sequence order, enforced by the sequence-number gate the previous chapter covers -- which means a given Queue Pair can never have more than one message open on its receiving side at once. A second FIRST cannot legitimately arrive before the first message's LAST closes it out, because that would require packets from two different messages to interleave on one Queue Pair's strictly ordered sequence, which Reliable Connection's own ordering guarantee does not allow. This is not a capacity limit a real implementation chose defensively -- it is a direct, structural consequence of the ordering guarantee [PSN Check and Out-of-Order Arrival](/rx/psn-check-and-out-of-order-arrival) already established, and it is exactly what keeps this chapter's state small and bounded: at most one open reassembly context per receiving Queue Pair, ever, never a working set that can grow under pressure the way a demand-driven cache can.

**What happens to this state when a message never closes.** A message opened by FIRST and never completed by a matching LAST -- because the sender stalled, because something downstream of this chapter's scope went wrong -- leaves this state open indefinitely unless something notices and reclaims it. No source this book found documents the specific mechanism a real implementation uses to detect and clean up an abandoned reassembly context under ordinary operation, distinct from the QP-reset case [Mid-Transaction RX Reset](/rx/mid-transaction-rx-reset) covers later in this part. This is an open question rather than a settled fact, named honestly rather than filled with a plausible-sounding guess.

> [!IMPORTANT] Food for Thought
> If you were adding a timeout to reclaim an abandoned reassembly context, what would you key it against -- wall-clock time since FIRST arrived, or something tied to the QP's own retry/timeout state in Part 5? What would a real implementation risk getting wrong either way?

**What happens next.** A packet that updates this chapter's state successfully -- landing at the address this context currently holds, advancing it correctly -- has done everything this part's ordering and addressing logic requires of it. What can still go wrong for that one Queue Pair specifically, and how that failure stays contained to it, is [Single-QP RX Stalls](/rx/single-qp-rx-stalls)'s subject, immediately next.

> [!TIP] Why
> Carrying only an address and a remaining count forward, rather than re-resolving the destination from RETH-equivalent information on every packet, is what makes a multi-packet message's continuation packets cheap to process: a MIDDLE packet does the minimum work a packet with no addressing information of its own could possibly need -- advance a pointer, place bytes, decrement a counter -- instead of repeating a resolution FIRST already paid for once. The cost of that design lands entirely on correctness discipline rather than on silicon: this state has to be seeded correctly from FIRST and updated correctly on every packet after it, because nothing downstream re-derives or double-checks the running address against RETH a second time. A single dropped or misordered update would place a later packet's payload at the wrong address with nothing else in this part positioned to catch it -- which is exactly why the sequence-number gate the previous chapter enforces has to hold perfectly before this chapter's state is ever touched.

[Single-QP RX Stalls](/rx/single-qp-rx-stalls) is next: what actually stalls one Queue Pair's receive path, and whether that stall touches anything beyond the Queue Pair that caused it.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- defines RETH's FIRST/ONLY-only placement and the ordering guarantee this chapter's boundedness argument rests on.
