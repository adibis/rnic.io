---
title: "The Send Queue Ring"
weight: 2
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "Posting a Work Request doesn't move a single byte -- it writes a WQE into a ring in host memory and rings a doorbell. This chapter is the ring itself: how it wraps around without a modulo, how a shared index survives being touched by both an untrusted process and a privileged hardware pipeline, and the one optimization that skips the ring's own memory entirely."
prev: /tx/the-wqe-and-the-send-queue-descriptor
next: /tx/wqe-prefetch
---

## Overview

A Send Queue's WQEs live in a ring in host memory -- a fixed-size, circular buffer software writes into and hardware reads from, tracked by a producer index (software's next write position) and a consumer index (hardware's next read position). Posting a Work Request writes a WQE into the ring at the producer index and advances it; it does not, by itself, move any data. A separate act -- ringing the doorbell -- is what actually tells hardware new work exists.

## Locator

![The master RNIC block diagram with Host Memory / Descriptor Rings and WQE Fetch / TX Scheduler both highlighted -- this chapter covers the ring itself and the doorbell that tells the fetch stage it has new work.](/images/tx/02-the-send-queue-ring-locator.svg)

[The WQE and the Send Queue Descriptor](../the-wqe-and-the-send-queue-descriptor) covered what one entry contains. This chapter covers the structure holding every entry for one Send Queue, and how hardware learns a new one exists.

## Block diagram

![An 8-slot Send Queue ring wrapping back to slot 0, with a consumer index and producer index each pointing into the ring and the slots between them shaded amber as posted-but-unfetched, captioned with the N-1 usable-capacity rule; below it, two notification paths to WQE Fetch -- an ordinary producer-index write through a 64-byte MMIO doorbell register, and a dashed BlueFlame path writing WQE bytes directly into a UAR buffer, bypassing the ring entirely for lower latency on small messages.](/images/tx/02-the-send-queue-ring-block.svg)

## Deep walkthrough

**Ring structure -- addressed by index, wrapped by masking, not modulo.** A real implementation's private tracking state for a ring like this needs, at minimum, a buffer pointer and size, an element size, and a way to wrap an index back to the start of the ring once it runs off the end. Constraining the ring's slot count to a power of two lets that wrap happen as a bitwise AND against a fixed mask rather than a division or modulo operation -- `(index + 1) & index_mask` -- a real, checkable optimization technique available to any implementation willing to accept that one constraint on ring sizing, not a claim about how every RNIC's ring is built.

**The off-by-one every ring like this needs.** A ring where the producer index equals the consumer index is defined as *empty* -- which means a ring with N physical slots can only ever hold N-1 posted-but-unconsumed entries, because filling all N slots would make the producer index wrap around and equal the consumer index again, indistinguishable from empty. This is not a bug or a wasted slot; it is the standard resolution every fixed-size ring with this exact ambiguity needs.

> [!NOTE] Concept
> A ring with N physical slots can only ever hold N-1 posted-but-unconsumed entries. Filling all N would make producer-equals-consumer mean both "full" and "empty" at once -- the one slot given up is what keeps that ambiguity from ever arising.

**Producer and consumer indices are shared, cross-privilege state, with synchronization consequences that follow directly.** The producer index lives in memory a userspace process writes directly (kernel-bypass, the same fact [What Is RDMA](/overview/what-is-rdma) opened this book with), while the consumer index is hardware's own. Reads and writes to this shared state need an explicit memory-ordering discipline between the two sides -- acquire/release semantics on each side of the handoff, not a full lock, because a lock would reintroduce exactly the per-operation kernel involvement RDMA's whole model exists to remove. This is a concrete instance of a general problem: any structure touched by both an untrusted, unsynchronized userspace process and a privileged intermediary or a hardware pipeline needs an explicit ordering discipline, not just "the data is correct because nothing else is running right now."

**Ringing the doorbell -- a measured mechanism, not a black box.** Kong et al. (NSDI 2023) measured the concrete shape of this step directly: "To post a work request, users need to ring the RNIC's doorbell through memory-mapped IO (MMIO). Each MMIO has a fixed aligned size 64-byte." The same paper traces the full path from there: the doorbell write reaches the adapter, which fetches the WQE the doorbell referenced from host DRAM, resolves whatever context it needs (this QP's live state, an SGE's `lkey`) against on-chip caches -- falling back to a DRAM fetch on a miss, exactly the cost [QP Caching](/qp/qp-caching) and [Key Caching](/mr/key-caching) already quantified -- and only then proceeds to build a packet. The same paper's own hardware diagram names a "TX Pipeline (Processing Unit)" and an "RX Pipeline (Processing Unit)" as separate blocks sharing common cache resources between them -- an independent, measured source corroborating this book's own master-diagram split between the two.

{{% details title="Aside: BlueFlame -- a named, non-standard optimization" closed="true" %}}
Mellanox's BlueFlame mechanism lets software write a WQE's bytes directly into a UAR (User Access Region -- device memory mapped for direct userspace access) in the same operation as ringing the doorbell, rather than writing to the ring and separately notifying hardware. Copying those bytes safely takes real care: the copy has to use instructions that bypass the CPU's own cache and preserve write ordering, because an ordinary memory copy can reorder the underlying writes in ways that don't matter when nothing else is reading the destination, but matter a great deal when hardware may be reading these bytes as they arrive rather than after a guaranteed-complete copy. What's public: the general concept (WQE bytes travel with the notification instead of a separate index update) and the existence of a doorbell record tracking a posted-WQE counter. What's not public: current-generation register-level layout, and whether every vendor's equivalent mechanism works identically. Treat BlueFlame as a real example of "notification plus payload in one write" going one step beyond a plain producer-index doorbell, not as the one universal mechanism every RNIC uses. Nor as a mechanism that scales to every WQE size: writing an entire WQE via non-temporal MMIO stores costs real CPU-side work per byte, and no source this book found states the specific size ceiling above which a real implementation stops routing a WQE through BlueFlame and falls back to the ordinary ring -- only that the technique is real, and that "small" is doing unquantified work in that sentence.
{{% /details %}}

**What happens next.** A doorbell ring (BlueFlame or ordinary) is where this chapter's job ends and the fetch side begins: getting that referenced WQE, or several WQEs ahead of when they're strictly needed, actually resident on-chip is [WQE Prefetch](/tx/wqe-prefetch)'s subject next.

> [!TIP] Why
> A ring buffer addressed by masking rather than modulo, and an off-by-one capacity rule that never lets producer and consumer collide ambiguously, are both instances of the same design principle: pay a small, fixed, one-time cost (one wasted slot; a power-of-two size constraint) to make every subsequent operation cheap and branch-free, rather than handling the ambiguous case explicitly on every single push and pop. BlueFlame makes a related trade at a different layer: writing WQE bytes directly into a UAR alongside the doorbell costs more CPU-side work per post (a larger MMIO write, non-temporal stores to bypass cache) in exchange for skipping the ring's own host-memory round trip on the hardware side for small, latency-sensitive messages -- a trade that only pays off when the message is small enough that the extra CPU-side copy is cheaper than the PCIe fetch it avoids, which is exactly why it exists as an optional path rather than replacing the ordinary ring entirely.

[WQE Prefetch](/tx/wqe-prefetch) is next: what happens between a doorbell ring and a WQE actually being on-chip and ready for the pipeline to build a packet from.

## References

- No IETF or IBTA standard governs Send Queue ring internals, doorbell register layout, or vendor-specific write-combining mechanisms like BlueFlame -- these are implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above, not a standards document.
