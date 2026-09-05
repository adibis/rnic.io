---
title: "The Send Queue Ring"
weight: 2
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "Posting a Work Request doesn't move a single byte -- it writes a WQE into a ring in host memory and rings a doorbell. This chapter is the ring itself: how it wraps around without a modulo, how a shared index survives being touched by both an untrusted process and a kernel driver, and the one optimization that skips the ring's own memory entirely."
prev: /tx/the-wqe-and-the-send-queue-descriptor
next: /tx/wqe-prefetch
---

## Locator

![The master RNIC block diagram with Host Memory / Descriptor Rings and WQE Fetch / TX Scheduler both highlighted -- this chapter covers the ring itself and the doorbell that tells the fetch stage it has new work.](/images/tx/02-the-send-queue-ring-locator.svg)

[The WQE and the Send Queue Descriptor](../the-wqe-and-the-send-queue-descriptor) covered what one entry contains. This chapter covers the structure holding every entry for one Send Queue, and how hardware learns a new one exists.

## Overview

A Send Queue's WQEs live in a ring in host memory -- a fixed-size, circular buffer software writes into and hardware reads from, tracked by a producer index (software's next write position) and a consumer index (hardware's next read position). Posting a Work Request writes a WQE into the ring at the producer index and advances it; it does not, by itself, move any data. A separate act -- ringing the doorbell -- is what actually tells hardware new work exists.

## Block diagram

<!-- IMAGE: A circular ring buffer drawn as a row of numbered slots wrapping visually back to slot 0, with a producer-index arrow and consumer-index arrow both pointing into it, the region between them shaded to represent posted-but-not-yet-fetched WQEs. A callout box reading "capacity = N slots, but only N-1 usable -- producer == consumer must mean empty, not ambiguous with full" sits beside the ring. Below it, a separate small doorbell sequence: an actor box "software" with two paths -- (a) a normal path: small "producer index write" box arrowing into a "doorbell register (MMIO, 64B aligned)" box, arrowing into "WQE Fetch" reading the ring; (b) a BlueFlame path drawn in a different color, labeled "WQE bytes written directly," bypassing the ring entirely with a dashed arrow straight into a "UAR (BlueFlame buffer)" box that feeds WQE Fetch without a separate ring read, captioned "lower latency for small messages -- skips the round trip through host-memory ring and back." -->

## Deep walkthrough

**Ring structure -- addressed by index, wrapped by masking, not modulo.** The Linux kernel's software RoCE driver (`rxe`) gives a real, checkable example of what a ring's private tracking state actually holds: a buffer pointer and size, an element size, and an `index_mask` used to wrap the index with a bitwise AND rather than a modulo operation -- `(index + 1) & index_mask` -- which only works because the ring's slot count is a power of two, a real, concrete design constraint worth naming rather than leaving implicit. This is `rxe`'s own implementation, not a claim that every RNIC's ring uses masking specifically, but it is a real, shipped example of the general technique, in the same spirit [The QP Table](/qp/the-qp-table) used `rxe` for wire-level detail elsewhere in this book.

**The off-by-one every ring like this needs.** A ring where the producer index equals the consumer index is defined as *empty* -- which means a ring with N physical slots can only ever hold N-1 posted-but-unconsumed entries, because filling all N slots would make the producer index wrap around and equal the consumer index again, indistinguishable from empty. This is not a bug or a wasted slot; it is the standard resolution every fixed-size ring with this exact ambiguity needs, and `rxe`'s own driver comment states this capacity rule directly.

**Producer and consumer indices are shared, cross-privilege state, and that has real synchronization consequences.** The producer index lives in memory a userspace process writes directly (kernel-bypass, the same fact [What Is RDMA](/overview/what-is-rdma) opened this book with), while the consumer index is hardware's own. `rxe`'s driver synchronizes reads and writes to this shared state with `smp_load_acquire()`/`smp_store_release()` -- acquire/release memory ordering, not a full lock, because a lock would reintroduce exactly the per-operation kernel involvement RDMA's whole model exists to remove. This is a real, concrete instance of a general problem: any structure touched by both an untrusted, unsynchronized userspace process and a privileged driver or a hardware pipeline needs an explicit ordering discipline, not just "the data is correct because nothing else is running right now."

**Ringing the doorbell -- a real, measured mechanism, not a black box.** Kong et al. (NSDI 2023) measured the concrete shape of this step directly: "To post a work request, users need to ring the RNIC's doorbell through memory-mapped IO (MMIO). Each MMIO has a fixed aligned size 64-byte." The same paper traces the full path from there: the doorbell write reaches the adapter, which fetches the WQE the doorbell referenced from host DRAM, resolves whatever context it needs (this QP's live state, an SGE's `lkey`) against on-chip caches -- falling back to a DRAM fetch on a miss, exactly the cost [QP Caching](/qp/qp-caching) and [Key Caching](/mr/key-caching) already quantified -- and only then proceeds to build a packet. The same paper's own hardware diagram names a "TX Pipeline (Processing Unit)" and an "RX Pipeline (Processing Unit)" as genuinely separate blocks sharing common cache resources between them -- a real, independent, measured source corroborating this book's own master-diagram split between the two.

**BlueFlame -- a real, named optimization that skips the ring's own round trip for small messages.** Mellanox's BlueFlame mechanism lets software write a WQE's bytes directly into a UAR (User Access Region -- device memory mapped for direct userspace access) in the same operation as ringing the doorbell, rather than writing to the ring and separately notifying hardware. The real, open-source `libmlx5` driver implementation copies WQE bytes using non-temporal store instructions specifically to bypass CPU caches, with the driver's own code comment warning against an ordinary `memcpy()` here because "memcpy() implementations may use move-string-buffer assembler instructions, which do not guarantee order of copying" -- ordering matters because hardware may be reading these bytes as they arrive, not after a guaranteed-complete copy. What's real and public: the general concept (WQE bytes travel with the notification instead of a separate index update), the existence of a doorbell record tracking a posted-WQE counter, and this specific driver's real implementation. What's not public: current-generation register-level layout, and whether every vendor's equivalent mechanism works identically. Treat BlueFlame as a real, documented example of "notification plus payload in one write" going one step beyond a plain producer-index doorbell, not as the one universal mechanism every RNIC uses.

**What happens next.** A doorbell ring (BlueFlame or ordinary) is where this chapter's job ends and the fetch side begins: getting that referenced WQE, or several WQEs ahead of when they're strictly needed, actually resident on-chip is [WQE Prefetch](/tx/wqe-prefetch)'s subject next.

> **Design rationale.** A ring buffer addressed by masking rather than modulo, and an off-by-one capacity rule that never lets producer and consumer collide ambiguously, are both instances of the same design principle: pay a small, fixed, one-time cost (one wasted slot; a power-of-two size constraint) to make every subsequent operation cheap and branch-free, rather than handling the ambiguous case explicitly on every single push and pop. BlueFlame makes a related trade at a different layer: writing WQE bytes directly into a UAR alongside the doorbell costs more CPU-side work per post (a larger MMIO write, non-temporal stores to bypass cache) in exchange for skipping the ring's own host-memory round trip on the hardware side for small, latency-sensitive messages -- a trade that only pays off when the message is small enough that the extra CPU-side copy is cheaper than the PCIe fetch it avoids, which is exactly why it exists as an optional path rather than replacing the ordinary ring entirely.

## Interface / timing reference

| Property | Value | Source / notes |
|---|---|---|
| Ring wraparound | bitwise mask, `(index + 1) & index_mask` | `rxe_queue.h`, requires a power-of-two slot count |
| Usable capacity | N-1 of N physical slots | producer == consumer is defined as empty |
| Index synchronization | `smp_load_acquire()` / `smp_store_release()` | `rxe`, shared between untrusted userspace and kernel/hardware |
| Doorbell MMIO size | 64 bytes, fixed and aligned | measured (Kong et al., NSDI 2023) |
| BlueFlame | WQE bytes written directly to UAR via non-temporal stores, alongside doorbell | `libmlx5`, real driver source -- vendor-specific, not universal |
| Doorbell record | posted-WQE counter, updated separately from BlueFlame's data copy | `libmlx5` (`qp->gen_data.db[MLX5_SND_DBR]`) |

[WQE Prefetch](/tx/wqe-prefetch) is next: what happens between a doorbell ring and a WQE actually being on-chip and ready for the pipeline to build a packet from.

## References

- No IETF or IBTA standard governs Send Queue ring internals, doorbell register layout, or vendor-specific write-combining mechanisms like BlueFlame -- these are implementation-specific silicon and driver design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above, not a standards document.
