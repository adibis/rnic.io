---
title: "Single-Burst Fault Handling"
weight: 7
date: 2026-09-16
publishDate: 2026-09-16
draft: false
description: "Every fault this book has named so far for a remote memory access happens before a transfer starts -- a bad key, an out-of-range address, a permission this operation never had. This chapter is about the narrower case left over: a transfer that was correctly authorized, already running, and still can't finish."
prev: /dma/ordering-and-outstanding-dma-tracking
next: /dma/multi-qp-burst-arbitration
metaDescription: "A DMA transfer that was authorized and started but cannot finish: an ODP page fault that pauses it, and a host-memory error that aborts it mid-buffer."
seoTitle: "DMA Fault Handling Mid-Transfer"
---

## Overview

[Address Translation Handoff](/dma/address-translation-handoff) already drew the line precisely: a bad key, an out-of-range address, or a disallowed operation type is [Permission and Bounds Violations](/mr/permission-and-bounds-violations)'s fault, caught before this engine ever begins moving bytes. What's left, once a transfer clears that gate and actually starts running, is narrower and genuinely this engine's own: one expected, recoverable stall, and one real, unrecoverable fault.

## Locator

![The master RNIC block diagram with DMA Engine highlighted and every other block dimmed -- this chapter covers what happens to an already-authorized transfer that can't proceed, not the authorization check itself.](/images/dma/07-single-burst-fault-handling-locator.svg)

## Block diagram

![A running transfer forking on what it hits: an On-Demand-Paging first-touch, pausing to cooperate with the OS's own page-fault resolution before resuming exactly where it left off, versus an uncorrectable host-memory access failure, aborting the transfer outright with whatever bytes already landed left in place, unrolled by nothing, and reported as a new fault class distinct from Single-QP Error Handling's two packet-triggered categories.](/images/dma/07-single-burst-fault-handling-block.svg)

{{< flow "dma-fault" >}}

## Deep walkthrough

**The expected case first: an On-Demand-Paging first touch is a stall, not a fault.** [Registering a Memory Region](/mr/registering-a-memory-region) already covers this precisely -- an ODP-registered range defers page pinning, and "the first access to a given page triggers a page-fault-style resolution, handled cooperatively by hardware and the OS against the OS's own page tables." When this engine's own gather or scatter reaches such a page for the first time, that resolution is what runs. [In-Flight Transfer State](/dma/in-flight-transfer-state)'s current address and remaining length are exactly what let the transfer resume where it paused rather than restart -- the same state this chapter's genuine-fault case, below, has no equivalent recovery for.

> [!CAUTION] Gotcha
> Don't file an ODP first-touch stall under this chapter's fault reporting. It's a real pause, and a real latency cost -- but it ends in the transfer completing successfully, the same as it would have if the page had already been resident. Nothing about it produces an error completion.

**The real fault: a host-memory access that simply fails.** An authorized, already-translated transfer can still fail on the memory side -- an uncorrectable error, or a memory-fabric access that never returns a good result. This is the one genuinely new fault class this Part contributes. [Single-QP Error Handling](/qp/single-qp-error-handling) names three triggers in total, but only two are packet-triggered and comparable to this one: an invalid-request error (a wire-level violation in an arriving packet) and an access-violation error (a bad key, bounds, or permission, caught before execution) -- both about what a packet asked for. Its third, a QP-fatal event cascading from a shared Completion Queue or Shared Receive Queue's own fault, is a different shape entirely -- not a property of any one operation, but a consequence one QP's own error inflicts on every other QP sharing that resource -- and this comparison isn't reaching for it. What this chapter adds is narrower than any of the three: whether the memory this engine was correctly asked to touch actually cooperated once execution began -- a distinct failure surface, local to this side's own memory subsystem, with no wire-level or permission-level cause at all.

IBTA-V1's own fault taxonomy has a real, if not perfectly named, home for this: Table 58's "Local QP Error" (Responder Class A, reported as NAK-Remote Operational Error) is defined in general terms broad enough to cover it -- "a local QP related error while executing the request message... [that] prevented the responder from completing the request." That general clause isn't scoped to any one cause; the specification's own concrete example under that heading is a receive queue failing to fetch a WQE from a Shared Receive Queue, not a host-memory access failure. This chapter's classification is therefore a plausible fit against the general definition, not a confirmed instance of the specification's own named example -- worth stating precisely rather than either overclaiming a direct citation or, as an earlier draft of this chapter did, understating that any source addresses this at all.

**No rollback -- a partial transfer stays partial.** Nothing this book found describes an undo mechanism for bytes a burst already placed before the fault that ends it. The honest, reasoned position: a multi-segment transfer that fails partway through leaves whatever it already wrote in place, unrolled by nothing, and reports failure for the operation as a whole regardless of how much of it actually succeeded. An application that needs to know precisely how much landed before a mid-burst failure has no completion-status field this book found that tells it -- the completion says the operation failed, not where.

> [!IMPORTANT] Food for Thought
> A multi-segment gather or scatter that fails on its third segment out of six has already moved data for the first two. What would it cost a real implementation to report exactly how far a failed transfer got, rather than a flat failure -- and is any application actually in a position to use that information usefully, given it would still have to treat the whole operation as failed either way?

**Confirmed, conditional on that classification: this fault forces the owning QP to ERROR.** [Single-QP Error Handling](/qp/single-qp-error-handling)'s flush-and-drain obligation exists precisely so a faulted operation always terminates with a real completion rather than a hang, and if this fault is indeed IBTA-V1's Local QP Error, Table 59 removes the guesswork: Responder Class A's own compliance behavior is "current receive WQE completed in error, subsequent flushed, queue to error state" -- the identical shape [Single-QP Error Handling](/qp/single-qp-error-handling) already describes for its own two packet-triggered classes. What remains genuinely unconfirmed is only the classification itself, not what happens once it's made -- the QP-to-ERROR consequence is mandatory for Local QP Error however this specific memory-side condition ends up counted.

**Where this connects.** A faulted transfer's completion is [CQE Generation](/cq/cqe-generation)'s to write, carrying whatever error status this class resolves to. What happens when this fault hits a burst attributed to a QP that's simultaneously being reset out from under it -- a narrower, timing-specific version of the same underlying question -- is [Mid-Transaction Burst Teardown](/dma/mid-transaction-burst-teardown)'s subject, not this chapter's.

## Interface reference

| Condition | Shape | Recoverable? |
|---|---|---|
| **On-Demand-Paging first touch** | expected page-fault-style resolution, cooperative with the OS | yes -- transfer resumes and completes normally |
| **Host-memory access failure** | a new local fault class, distinct from Single-QP Error Handling's two packet-triggered categories (its third, QP-fatal, is a differently-shaped cascade, not part of this comparison) | no -- transfer aborts, partial writes are not undone |
| **QP-level consequence** | classification as IBTA-V1's Local QP Error is plausible, not confirmed by name; the ERROR/flush outcome itself is mandatory (Table 59, Responder Class A) once that classification holds | QP transitions to ERROR, flush-and-drain applies |

## References

- [Registering a Memory Region](/mr/registering-a-memory-region) and its RDMA man-pages citation for On-Demand Paging's first-touch behavior -- reused here, not re-derived.
- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- Table 58's "Local QP Error" (Responder Class A, NAK-Remote Operational Error) is defined broadly enough to plausibly cover this chapter's host-memory-execution fault class, and Table 59 confirms its mandatory QP-to-ERROR, flush-and-drain consequence. The specification's own only named example under that heading is a Shared Receive Queue WQE-fetch failure, not a host-memory access failure -- this chapter's classification is a plausible fit against the general definition, not a confirmed instance of the named example, and is stated with that precision rather than as either an unsourced claim or a settled one.
