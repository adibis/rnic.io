---
title: "The Scatter Path"
weight: 5
date: 2026-09-16
publishDate: 2026-09-16
draft: false
description: "Every arriving payload lands in memory this side can address directly -- the only question is who decided where. A RECEIVE lands where this side's own software already posted a buffer. A WRITE lands where the wire packet itself said to. This chapter is the mechanism RX Pipeline has been calling DMA writeback all along, given its own name and its own depth."
prev: /dma/the-gather-path
next: /dma/ordering-and-outstanding-dma-tracking
metaDescription: "How an RNIC writes an arriving payload to memory: across the Receive WQE's SGE list for a SEND, or at the RETH address for an RDMA WRITE."
seoTitle: "The RDMA Scatter Path"
---

## Overview

Scatter is this engine's RX-side job: writing an arriving payload into memory, one already-translated segment at a time. Every scatter this engine ever runs writes into memory this side can address directly -- there's no such thing as writing into memory that's actually remote from the engine doing the writing. What varies, per [Target Selection](/dma/target-selection), is who chose the address: this side's own software, ahead of time, or the packet that just arrived.

> [!NOTE] Concept
> This is the mechanism [PSN Check and Out-of-Order Arrival](/rx/psn-check-and-out-of-order-arrival) already named, without describing, as "DMA writeback" -- the same handoff, given its own chapter here rather than left as a forward reference.

## Locator

![The master RNIC block diagram with DMA Engine highlighted and every other block dimmed -- this chapter is the RX-side half of this engine's two data-movement paths, fed by RX PSN Check's own handoff.](/images/dma/05-the-scatter-path-locator.svg)

## Block diagram

![Four trigger cases -- RECEIVE, RDMA WRITE, RDMA WRITE with Immediate, an RDMA READ response's arrival, and an Atomic's return value -- each writing into a destination resolved through Address Translation Handoff, software-named for RECEIVE and both requester-side cases, wire-named for WRITE; WRITE with Immediate drawn with two separate arrows, one into the RETH-named payload destination and a second into a consumed RQ WQE that never receives the payload itself.](/images/dma/05-the-scatter-path-block.svg)

## Deep walkthrough

**RECEIVE -- writing where this side's own software already said to.** A SEND arriving at this QP is placed into whichever Receive Queue Element this side posted ahead of time, in FIFO order against however many are queued -- software-named, resolved through that WQE's own SGE list and `lkey`, exactly as [Target Selection](/dma/target-selection) already established.

**RDMA WRITE -- writing where the packet itself said to.** No Receive Queue Element is consulted at all. The destination comes from RETH, carried in the packet, resolved against the `rkey` it names -- this side's software had no role in choosing this address at the moment the packet arrived, even though the memory it lands in is registered, in advance, by that same software.

**RDMA WRITE with Immediate -- two separate destinations, easy to conflate.** The payload itself scatters exactly like a plain WRITE, wire-named via RETH. Separately, this variant consumes one posted Receive Queue Element -- not to place the payload, only to give the immediate value somewhere to attach to and to produce a receive completion the application can poll. Treat these as two independent writes with two independent destinations, not one write with two names.

> [!CAUTION] Gotcha
> Don't assume a WRITE with Immediate's payload ends up in the consumed RQ WQE's own buffer. It doesn't -- that buffer receives nothing but the immediate value's bookkeeping. The payload itself goes exactly where a plain WRITE's would, at the RETH-named address, entirely independent of whichever RQ WQE happened to be next in line.

**RDMA READ response -- writing back into the request that asked for it.** The requester's own DMA engine scatters an arriving response payload into the local buffers its original request WQE named -- software-named, the same SGE list [Target Selection](/dma/target-selection) already pointed at. [In-Flight Transfer State](/dma/in-flight-transfer-state)'s completion tag is what makes this possible across a real network round trip: the request's own destination identity has to survive from the moment it was issued until whenever the response actually arrives, which could be many other packets later.

**Atomics -- one word, not a list.** The pre-operation value an atomic's acknowledgement carries back -- [The Operation Catalog](/overview/operation-catalog/atomics) already names this precisely, ATMACK's "returns the pre-operation value," not the result of the operation -- lands in a single local buffer the original WQE named. No scatter-gather cursor applies; an atomic touches exactly one location on each side, never a multi-segment chain.

**Where scattering sits relative to the sequence-number gate.** [PSN Check and Out-of-Order Arrival](/rx/psn-check-and-out-of-order-arrival) already establishes that a packet has to clear sequence-number ordering before its payload is safe to place at all -- an out-of-order arrival is rejected and resent later, never scattered early and reconciled afterwards. This path never runs ahead of that gate; by the time this chapter's mechanism starts, the packet it's serving is already known to be the next one this QP's receive side expects.

The walk below receives that same SEND into a Receive WQE with a 2 KiB and an 8 KiB buffer, so the first packet splits across both and the second resumes inside the larger one.

{{< flow "sge-scatter" >}}

**Where this connects.** A completed scatter is what [CQE Generation](/cq/cqe-generation) is waiting on before it can write a receive-side completion -- unconditionally, per that chapter's own asymmetric rule for receive-side events. What happens if a scatter can't complete -- the destination faults partway through a multi-segment write -- is [Single-Burst Fault Handling](/dma/single-burst-fault-handling)'s question, not this chapter's.

## Interface reference

| Trigger opcode | Destination | Addressing (per Target Selection) |
|---|---|---|
| **RECEIVE** (SEND arriving) | receiver's posted RQ WQE SGE list | software-named |
| **RDMA WRITE** | RETH virtual address, `rkey` | wire-named |
| **RDMA WRITE w/ Immediate** | payload: RETH address (wire-named); RQ WQE consumed only for immediate + completion | mixed, see Gotcha above |
| **RDMA READ response** | requester's original WQE SGE list | software-named |
| **Atomic acknowledgement** | requester's original WQE buffer (single location, pre-operation value) | software-named |

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 -- governs RETH, ATMACK, and the receive-completion semantics this chapter's scatter destinations are derived from, already cited in full in [The Operation Catalog](/overview/operation-catalog).
- Linux kernel soft-RoCE driver, `drivers/infiniband/sw/rxe/` -- `rxe_resp.c` places a SEND's payload with the same `copy_data()` cursor walk over the Receive WQE's SGE list, checking local-write access per `lkey`; running out of SGEs returns `-ENOSPC`, which an RC QP answers with an invalid-request NAK rather than truncating.
