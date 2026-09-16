---
title: "Single-Burst Fault Handling"
weight: 7
date: 2026-09-16
publishDate: 2026-09-16
draft: false
description: "Every fault this book has named so far for a remote memory access happens before a transfer starts -- a bad key, an out-of-range address, a permission this operation never had. This chapter is about the narrower case left over: a transfer that was correctly authorized, already running, and still can't finish."
prev: /dma/ordering-and-outstanding-dma-tracking
next: /dma/multi-qp-burst-arbitration
---

## Overview

[Address Translation Handoff](/dma/address-translation-handoff) already drew the line precisely: a bad key, an out-of-range address, or a disallowed operation type is [Permission and Bounds Violations](/mr/permission-and-bounds-violations)'s fault, caught before this engine ever begins moving bytes. What's left, once a transfer clears that gate and actually starts running, is narrower and genuinely this engine's own: one expected, recoverable stall, and one real, unrecoverable fault.

## Locator

![The master RNIC block diagram with DMA Engine highlighted and every other block dimmed -- this chapter covers what happens to an already-authorized transfer that can't proceed, not the authorization check itself.](/images/dma/07-single-burst-fault-handling-locator.svg)

## Block diagram

![A running transfer forking on what it hits: an On-Demand-Paging first-touch, pausing to cooperate with the OS's own page-fault resolution before resuming exactly where it left off, versus an uncorrectable host-memory access failure, aborting the transfer outright with whatever bytes already landed left in place, unrolled by nothing, and reported as a new, third local fault class distinct from Single-QP Error Handling's existing two.](/images/dma/07-single-burst-fault-handling-block.svg)

## Deep walkthrough

**The expected case first: an On-Demand-Paging first touch is a stall, not a fault.** [Registering a Memory Region](/mr/registering-a-memory-region) already covers this precisely -- an ODP-registered range defers page pinning, and "the first access to a given page triggers a page-fault-style resolution, handled cooperatively by hardware and the OS against the OS's own page tables." When this engine's own gather or scatter reaches such a page for the first time, that resolution is what runs. [In-Flight Transfer State](/dma/in-flight-transfer-state)'s current address and remaining length are exactly what let the transfer resume where it paused rather than restart -- the same state this chapter's genuine-fault case, below, has no equivalent recovery for.

> [!CAUTION] Gotcha
> Don't file an ODP first-touch stall under this chapter's fault reporting. It's a real pause, and a real latency cost -- but it ends in the transfer completing successfully, the same as it would have if the page had already been resident. Nothing about it produces an error completion.

**The real fault: a host-memory access that simply fails.** An authorized, already-translated transfer can still fail on the memory side -- an uncorrectable error, or a memory-fabric access that never returns a good result. This is the one genuinely new fault class this Part contributes. [Single-QP Error Handling](/qp/single-qp-error-handling) already names two: a requester-side transport error (a wire-level violation in an arriving packet) and an access-violation error (a bad key, bounds, or permission, caught before execution). Both are about what a packet asked for. This one is about whether the memory this engine was correctly asked to touch actually cooperated once execution began -- a distinct failure surface, local to this side's own memory subsystem, with no wire-level or permission-level cause at all.

**No rollback -- a partial transfer stays partial.** Nothing this book found describes an undo mechanism for bytes a burst already placed before the fault that ends it. The honest, reasoned position: a multi-segment transfer that fails partway through leaves whatever it already wrote in place, unrolled by nothing, and reports failure for the operation as a whole regardless of how much of it actually succeeded. An application that needs to know precisely how much landed before a mid-burst failure has no completion-status field this book found that tells it -- the completion says the operation failed, not where.

> [!IMPORTANT] Food for Thought
> A multi-segment gather or scatter that fails on its third segment out of six has already moved data for the first two. What would it cost a real implementation to report exactly how far a failed transfer got, rather than a flat failure -- and is any application actually in a position to use that information usefully, given it would still have to treat the whole operation as failed either way?

**Reasoned, not confirmed: this fault almost certainly still forces the owning QP to ERROR.** [Single-QP Error Handling](/qp/single-qp-error-handling)'s flush-and-drain obligation exists precisely so a faulted operation always terminates with a real completion rather than a hang. A memory-side execution fault leaves an operation just as unable to complete normally as either of that chapter's two named causes -- the architecturally consistent expectation is that it drives the same QP-to-ERROR transition and the same flush behavior. No source this book found states this explicitly for this specific fault class; it's this book's own extension by analogy, flagged as such rather than asserted as a confirmed fact.

**Where this connects.** A faulted transfer's completion is [CQE Generation](/cq/cqe-generation)'s to write, carrying whatever error status this class resolves to. What happens when this fault hits a burst attributed to a QP that's simultaneously being reset out from under it -- a narrower, timing-specific version of the same underlying question -- is [Mid-Transaction Burst Teardown](/dma/mid-transaction-burst-teardown)'s subject, not this chapter's.

## Interface reference

| Condition | Shape | Recoverable? |
|---|---|---|
| **On-Demand-Paging first touch** | expected page-fault-style resolution, cooperative with the OS | yes -- transfer resumes and completes normally |
| **Host-memory access failure** | a new, third local fault class, distinct from Single-QP Error Handling's transport-error and access-violation categories | no -- transfer aborts, partial writes are not undone |
| **QP-level consequence** | reasoned by analogy to Single-QP Error Handling's existing two classes -- not independently confirmed | QP transitions to ERROR, flush-and-drain applies |

## References

- [Registering a Memory Region](/mr/registering-a-memory-region) and its RDMA man-pages citation for On-Demand Paging's first-touch behavior -- reused here, not re-derived.
- No IBTA or IETF source describes a local host-memory-execution fault class for a DMA engine -- this chapter's central claim is this book's own reasoned inference, stated plainly as such throughout.
