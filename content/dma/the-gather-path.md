---
title: "The Gather Path"
weight: 4
date: 2026-09-16
publishDate: 2026-09-16
draft: false
description: "Three opcodes need bytes read out of local memory before this adapter can send anything: a SEND building its own message, a WRITE building the payload the far end will place, and an RDMA READ response gathering exactly what the requester asked for. All three walk the same path -- and the payload has to actually be in hand before Packet Build can assemble anything around it."
prev: /dma/address-translation-handoff
next: /dma/the-scatter-path
metaDescription: "How an RNIC reads a payload out of local memory for a SEND, WRITE or READ response: walking the SGE list, one key lookup per segment, packet by packet."
seoTitle: "The RDMA Gather Path"
---

## Overview

Gather is this engine's TX-side job: reading local memory, one already-translated segment at a time, to assemble a payload before it leaves the port. Three opcodes trigger it -- SEND, RDMA WRITE, and an RDMA READ response's own construction at the responder -- and none of them can proceed to [TX Packet Build](/tx/tx-packet-build) until this path finishes.

> [!NOTE] Concept
> Gather and software-named addressing aren't the same idea, even though they usually travel together. SEND and WRITE's sources are software-named, per [Target Selection](/dma/target-selection). An RDMA READ response's source is a gather too -- reading this side's own memory to build an outgoing packet -- but the address it reads from is wire-named, carried in the original request's RETH. This chapter covers both.

## Locator

![The master RNIC block diagram with DMA Engine highlighted and every other block dimmed -- this chapter is the TX-side half of this engine's two data-movement paths, feeding into TX Packet Build.](/images/dma/04-the-gather-path-locator.svg)

## Block diagram

![Three trigger opcodes -- SEND, RDMA WRITE, RDMA READ response -- each walking a scatter-gather list one segment at a time, each segment resolved through Address Translation Handoff independently, assembled into one payload and handed to TX Packet Build; a separate, dashed inline path bypasses this entire chapter for payloads copied directly into the WQE at post time.](/images/dma/04-the-gather-path-block.svg)

## Deep walkthrough

**The inline fork happens before this chapter ever starts.** [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor) already covers this precisely: a payload within the inline threshold is copied directly into the WQE by software at post time, and never touches a scatter-gather entry or an `lkey` at all. This path exists only for what's left -- a payload that couldn't or wasn't requested to be inlined, and that this engine has to go fetch itself, "a mechanism [The Send Queue Ring](/tx/the-send-queue-ring) picks up from the fetch side and DMA Engine owns in full," in that chapter's own words.

**Each segment is its own lookup.** A scatter-gather entry carries its own address, length, and `lkey` independently -- [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor) already established that a single operation can span up to 63 data segments, each potentially naming a different Memory Region entirely. [In-Flight Transfer State](/dma/in-flight-transfer-state)'s scatter-gather cursor is what tracks which segment is active; [Address Translation Handoff](/dma/address-translation-handoff) gets called once per segment, not once per operation, since nothing guarantees consecutive segments even resolve through the same Key Cache entry.

**SEND and RDMA WRITE -- reading this side's own memory to build a message.** Both gather from a source their own software named at post time. The bytes this path reads become the message body a SEND carries, or the payload a WRITE's own RETH will tell the far end where to place -- this engine doesn't know or care which opcode it's serving at the byte-movement level; the opcode only decided which SGE list to walk.

**RDMA READ response -- gathering from wherever the request pointed, not from anything this side posted.** [Target Selection](/dma/target-selection) already named this the one case where gather and wire-named addressing coincide: the responder services a READ request by gathering from the local memory the *request's own* RETH named, not from a WQE this side's software wrote. The result is a response message this engine still has to assemble the same way it assembles any other outgoing payload -- gathered, one segment at a time, and hand it onward.

> [!TIP] Why
> Treating an RDMA READ response's construction as an ordinary gather, rather than inventing a separate "response-building" mechanism, is what lets this engine stay opcode-agnostic at the byte-movement level. Target Selection already decided which address and key apply; by the time this path runs, it's walking a resolved segment list exactly the same way regardless of whether a SEND's own software or a remote peer's READ request is what asked for these bytes.

**Where gathering sits relative to the rate limiter's gate.** [Rate-Limiter Context](/congestion/rate-limiter-context) already established that a rate-limited QP's dispatch decision happens "before the adapter spends any further effort on a WQE," and [The Rate Limiter Table](/congestion/the-rate-limiter-table) sharpens that to a specific point: "before a scheduled WQE is allowed to proceed toward becoming a packet" -- the same reasoning argues gathering should wait for that same gate, not run ahead of it. Fetching payload bytes for a transfer the rate limiter is about to hold serves no purpose and burns memory bandwidth a still-throttled QP has no immediate use for. This book states that ordering as the sensible design, not a confirmed cycle-accurate pipeline fact from any specific implementation.

> [!CAUTION] Gotcha
> Don't assume every gather this engine ever runs is TX-only in the sense of "outgoing traffic this local software originated." An RDMA READ response is outgoing, from this side's port, and it's still a gather this side's own software never asked for -- it exists because a remote peer's request demanded it. "Gather" describes a direction of data movement, not a claim about who initiated the operation.

The walk below gathers one 7,680-byte SEND from three buffers at a 4,096-byte path MTU. The first packet ends partway through the second buffer, and the cursor picks up from there for the next one.

{{< flow "sge-gather" >}}

**Where this connects.** A completed gather hands its assembled payload to [TX Packet Build](/tx/tx-packet-build), which wraps it in whatever header the opcode requires and computes ICRC over the result. What happens if a segment's own lookup fails partway through a multi-segment gather -- some bytes already fetched, the rest never will be -- is [Single-Burst Fault Handling](/dma/single-burst-fault-handling)'s question, not this chapter's.

## Interface reference

| Trigger opcode | Source | Addressing (per Target Selection) |
|---|---|---|
| **SEND** | sender's own WQE SGE list | software-named |
| **RDMA WRITE** | sender's own WQE SGE list | software-named |
| **RDMA READ response** | responder's own memory, at the address the request's RETH named | wire-named |
| **Atomics** | none -- operand is inline in the request packet itself | n/a, not this path |

## References

- No IBTA or IETF standard names a "gather path" as its own mechanism -- the opcode behavior above is derived from RETH and SGE-list semantics already covered field by field in [The Operation Catalog](/overview/operation-catalog) and [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor).
- Linux kernel soft-RoCE driver, `drivers/infiniband/sw/rxe/` -- `rxe_req.c` sizes each packet as the smaller of the path MTU and the bytes left in the message, and `copy_data()` in `rxe_mr.c` fills it by walking the WQE's SGE list from a saved cursor (`cur_sge`, `sge_offset`, `resid`), looking up each SGE's `lkey` as the cursor reaches it. The walk's per-packet model follows this driver.
