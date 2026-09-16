---
title: "Ordering and Outstanding-DMA Tracking"
weight: 6
date: 2026-09-16
publishDate: 2026-09-16
draft: false
description: "A completion notification is a weaker promise than it sounds like. IBTA guarantees a QP's own requests execute in order -- it does not guarantee a completed write is actually visible in memory yet, a gap real enough that a later IBTA extension added an entire new operation just to close it. Separately: this engine's own concurrent-transfer capacity is a resource no QP-level attribute already covers."
prev: /dma/the-scatter-path
next: /dma/single-burst-fault-handling
---

## Overview

Two independent questions share this chapter because they're both, at heart, about what a QP is entitled to assume while this engine is working on its behalf. The first is a standard-defined guarantee, precise but narrower than it sounds: what order does a QP's own requests execute in, and what does a completion actually promise about data having landed. The second has no standard at all: how many transfers can this engine actually run at once, across every QP calling into it, and what happens to the QP that finds that capacity already spoken for.

## Locator

![The master RNIC block diagram with DMA Engine highlighted and every other block dimmed -- this chapter covers what this engine's own execution guarantees and doesn't guarantee, and a resource this Part hasn't named yet: how many transfers it can run at once.](/images/dma/06-ordering-and-outstanding-dma-tracking-locator.svg)

## Block diagram

![Two panels: a per-QP request stream executing strictly in order on the left, with a completion arrow explicitly not reaching all the way to a "visible in memory" marker -- a gap bridged only by an optional FLUSH operation -- and, on the right, a shared pool of concurrent transfer slots drawn as a fixed-size bank, contended across several QPs, with one QP's request shown waiting because the pool is currently full.](/images/dma/06-ordering-and-outstanding-dma-tracking-block.svg)

## Deep walkthrough

**What IBTA actually guarantees for a Reliable Connection: in-order execution, not immediate visibility.** A QP's own posted requests execute in the order they were posted, and a signaled completion implicitly confirms every request ahead of it in that same order has also completed -- the same cumulative guarantee [Go-Back-N Retransmission](/reliability/go-back-n-retransmission) already relies on for its own retry logic. That's a real, standard-backed guarantee about *sequence*. It says nothing, on its own, about *visibility* -- whether a write's bytes have actually reached memory in a form a subsequent, unrelated read of that same location would observe.

> [!CAUTION] Gotcha
> "My completion fired, so my data is in memory" is a weaker claim than it sounds like. A completion for a WRITE or a SEND confirms the responder's *reliability engine* acknowledged the transfer -- it does not, by itself, confirm this engine's own scatter has finished landing bytes in the memory hierarchy behind that acknowledgement. Fast, posted writes into a memory subsystem can be absorbed into a buffer that accepts responsibility for placement, while reporting success to whatever asked immediately.

**Real enough a gap that IBTA built a fix for it.** IBTA's own Memory Placement Extensions (Volume 1, Release 1.5) added a new operation, FLUSH, for exactly this reason: it executes like an RDMA Read -- ordered, flow-controlled, acknowledged -- and guarantees that every prior write on that connection has actually become visible to the responder before FLUSH's own completion reaches the requester. An application with no durability requirement beyond ordinary completion semantics has no reason to reach for it; one that needs to know bytes are actually placed -- persistent-memory writes being the motivating case -- issues a FLUSH specifically because ordinary completion doesn't promise that on its own.

> [!NOTE] Concept
> Treat "completed" and "visible" as two separate claims about a write, not two words for the same fact. Ordinary completion promises the first. Only FLUSH promises the second.

**An explicit ordering request between a Read and what follows it.** A Queue Pair can carry an ordering flag on a request, honored by this engine, that forces it to wait for every preceding outstanding RDMA Read or Atomic on that same QP to finish before starting execution on the flagged request. This matters because a plain SEND or WRITE posted right after a READ has no *inherent* reason to wait for that READ's data to actually arrive -- without an explicit request otherwise, this engine is free to run them concurrently. This book states this mechanism functionally, as an architect's-choice execution-ordering behavior a real implementation provides on request, rather than a wire-visible or universally standardized artifact -- no source found ties a specific bit-level encoding to IBTA itself, and this book isn't going to assert one that isn't confirmed.

**Outstanding read/atomic tracking already has an owner -- and it isn't this chapter.** [QP Context](/qp/qp-context) already covers a real, IBTA-defined pair of fields: a requester-side ceiling bounding how many RDMA reads and atomics *this* QP may have outstanding at once, and a responder-side ceiling bounding how many it will accept from its peer. That's a per-QP negotiated attribute, set once at connection setup. It is not the resource this chapter's second half is about.

> [!WARNING] Analogy
> QP Context's ceiling is like a credit limit -- how much *this one account* is allowed to owe at once. This engine's own concurrent-transfer capacity, below, is like the teller window itself -- how many transactions the branch can physically process at the same moment, regardless of whose account any of them belong to. A QP well within its own credit limit can still be made to wait if every teller window is already busy serving someone else.

**This engine's own concurrent-transfer capacity -- a real resource, with no standard describing it.** However many transfers this engine's hardware can actually execute at once is a fixed, implementation-specific capacity, shared across every QP that calls into it -- the same shape of provisioning question as how many outstanding transactions a memory controller or a PCIe root port can track at once. [In-Flight Transfer State](/dma/in-flight-transfer-state) is what one occupied slot in that capacity looks like; this chapter is about the size of the pool those slots come from, and what happens when every slot is already spoken for. [Single-QP RX Stalls](/rx/single-qp-rx-stalls) already named the symptom directly -- "a DMA engine already occupied placing something else" -- without describing the resource behind it. [Multi-QP Burst Arbitration](/dma/multi-qp-burst-arbitration), immediately next, is what a QP's own request actually experiences when it finds that pool full.

> [!CAUTION] Gotcha
> Don't reach for QP Context's outstanding-read/atomic ceiling to explain a stalled DMA. That ceiling only ever limits one QP's own read/atomic pipeline depth against its own peer's stated capacity -- it says nothing about whether the shared engine executing an ordinary WRITE's scatter, or a SEND's gather, has a free slot to run it in right now.

## Interface reference

| Question | Answer |
|---|---|
| **Does a QP's own requests execute in order** | yes -- IBTA-guaranteed for RC, cumulative completion semantics apply |
| **Does a completion guarantee memory-hierarchy visibility** | no -- only FLUSH (IBTA Memory Placement Extensions) makes that guarantee explicitly |
| **Does this engine wait for a prior Read before a later Send/Write by default** | no, unless the request explicitly asks for that ordering |
| **What bounds one QP's own outstanding reads/atomics** | [QP Context](/qp/qp-context)'s requester-/responder-side ceiling pair -- not this chapter's resource |
| **What bounds how many transfers this engine runs at once, across all QPs** | an implementation-specific concurrent-transfer capacity -- no standard defines it |

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 -- governs RC's in-order execution and cumulative completion semantics.
- InfiniBand Trade Association, *Memory Placement Extensions*, Supplement to InfiniBand Architecture Specification Volume 1, Release 1.5 (2021) -- introduces the FLUSH operation and its explicit prior-writes-visible-before-completion guarantee, the standard-level closing of the gap this chapter's central Gotcha describes.
