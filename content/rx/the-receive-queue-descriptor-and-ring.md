---
title: "The Receive Queue Descriptor and Ring"
weight: 1
date: 2026-09-09
publishDate: 2026-09-09
draft: false
description: "A Send Queue entry tells hardware what to do. A Receive Queue entry tells hardware only where -- no opcode, no remote address, nothing to send. This chapter is the simplest descriptor in the book, and the ring that holds it in strict, unaddressed order."
next: /rx/rx-packet-parse
metaDescription: "An RDMA Receive WQE says only where data should land: no opcode, no remote address. The descriptor, and the ring that holds it in order."
seoTitle: "The RDMA Receive Queue and Receive WQE"
---

## Overview

A Receive Queue entry is the simplest descriptor this book has covered. Where a Send Queue entry names an operation -- an opcode, a scatter-gather list of source bytes, whatever remote-addressing fields that opcode needs -- a Receive Queue entry names only a destination: a scatter-gather list of buffers ready to receive whatever payload lands next. It carries no opcode, because it does not describe an operation; it carries no remote address, because nothing about receiving requires naming where the sender's bytes came from at the descriptor level. It says exactly one thing: here is where the next unaddressed arrival should go.

## Locator

![The master RNIC block diagram with Host Memory / Descriptor Rings and RX Parser / Reassembly both highlighted, mirroring TX Pipeline's Send Queue Ring locator on the opposite side of the pipeline -- this chapter covers what one Receive Queue entry contains and the ring holding it, not the parsing stage that consumes it.](/images/rx/01-the-receive-queue-descriptor-and-ring-locator.svg)

[One Transaction, Start to Finish](/overview/one-transaction-start-to-finish) already named a Receive Queue in passing -- a place software provisions buffers "ahead of expected traffic." This chapter opens that structure the same way [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor) opened the Send Queue's: what one entry actually contains, and the ring holding every entry for one Receive Queue.

## Block diagram

![A Receive Queue ring holding posted buffer descriptors (SGE list only, no opcode field, no remote-address field) between a producer index software advances and a consumer index hardware advances, with a doorbell path running in the reverse direction from TX Pipeline's Send Queue Ring diagram -- software telling hardware how many new buffers exist, not how much new work is ready.](/images/rx/01-the-receive-queue-descriptor-and-ring-block.svg)

## Deep walkthrough

**What the standard requires, and how thin that requirement is.** IBTA-V1 specifies a Receive Request's content at the same semantic level it specifies a Send Request's: a list of local memory segments, each an address, a length, and the key authorizing local access to it. Nothing else. It does not require an opcode field, because a Receive Request does not select an operation -- it is consumed by whichever arriving operation needs a buffer, not chosen in advance to match one. This is the architectural root of everything else in this chapter: a descriptor with no opcode cannot be addressed to a specific future packet, which is exactly why what holds it has to be a strict, unaddressed queue rather than anything a lookup could target.

**The ring, and why it has to be consumed in strict order.** A Receive Queue's entries live in a ring the same shape as a Send Queue's -- a fixed-size circular buffer, a producer index advanced by software as it posts new entries, a consumer index advanced by hardware as it consumes them. The direction of intent is reversed from the Send Queue's ring, though, and that reversal is the whole point of this structure: a Send Queue's doorbell tells hardware *work is ready to go out*; a Receive Queue's doorbell tells hardware *buffers are ready to be filled*. Because a Receive Queue entry carries no addressing information of its own, hardware has no basis for choosing which entry to consume for a given arrival -- it can only take the one at the head of the ring, in the exact order software posted it. This is not a design choice a real implementation is free to relax; it falls directly out of the descriptor's own poverty of information. A structure that could name which arrival it was for would not need this constraint, and would not be a Receive Queue entry as the standard defines one.

**Capacity, and the ambiguity every ring this shape has to resolve.** A ring where the producer index and the consumer index point to the same slot is ambiguous on its face -- it could mean the ring is completely empty, or completely full, and nothing in the two indices alone distinguishes the two. Every real implementation of a producer-consumer ring resolves this the same way: reserve one slot's worth of capacity as the tiebreaker, so a ring with N physical slots holds at most N-1 posted, unconsumed entries, and equal indices always means empty rather than sometimes meaning full. This is not a wasted slot in any meaningful sense -- it is the fixed, one-time cost of making every subsequent index comparison unambiguous, paid once at design time rather than on every single post.

**Provisioning ahead of need -- and what happens when provisioning falls behind.** Because nothing at post time associates a Receive Queue entry with any specific future arrival, software's whole job is keeping the ring stocked ahead of demand: post enough buffers, early enough, that an operation needing one never arrives to find the ring empty. What happens when that discipline slips -- an operation that needs a Receive Queue entry arriving to find none posted -- is a named failure mode this book has already flagged once, in the operation catalog's description of SEND's two-sided contract, and gets its full architectural treatment in [Single-QP RX Stalls](/rx/single-qp-rx-stalls) rather than here. This chapter's job stops at the structure itself; what a starved structure actually does is that chapter's subject.

**What actually consumes an entry, and where this chapter's job ends.** Popping the head of this ring is not something that happens on a timer or a guess -- it happens exactly when an arriving operation is confirmed to need one, a determination [RX Packet Parse](/rx/rx-packet-parse) and [PSN Check and Out-of-Order Arrival](/rx/psn-check-and-out-of-order-arrival) make together before this ring is ever touched. This chapter covers the structure that sits ready to be consumed; what triggers the consumption, and what happens to the bytes that land in the buffer it names, belongs to the chapters immediately following.

> [!TIP] Why
> Stripping a Receive Queue entry down to nothing but a destination, with no opcode and no addressing, is what makes strict FIFO consumption both correct and cheap: correct, because an entry that could name which future arrival it belongs to would need some matching logic hardware doesn't have any basis to run before a packet arrives; cheap, because popping the head of a ring is a fixed, branch-free operation regardless of how deep the ring is, where any kind of targeted lookup would cost more silicon for a case this descriptor's own design makes unnecessary. The cost of that simplicity lands entirely on software's side of the boundary: keeping the ring stocked ahead of demand is now software's problem to solve, not hardware's problem to hide, and a Receive Queue that runs dry is not a hardware fault -- it is exactly the failure mode this simple a descriptor should be expected to produce when the discipline around it slips.

[RX Packet Parse](/rx/rx-packet-parse) is next: what actually happens to an arriving frame before anything in this chapter's ring is ever touched.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
