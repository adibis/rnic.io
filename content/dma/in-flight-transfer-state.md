---
title: "In-Flight Transfer State"
weight: 1
date: 2026-09-16
publishDate: 2026-09-16
draft: false
description: "Every per-QP context this book has covered so far survives between packets -- installed once, held, updated in place. This engine keeps nothing like that. What it tracks exists only for the lifetime of one transfer, belongs to whichever QP asked for it, and disappears the instant that transfer completes."
next: /dma/target-selection
---

[Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) already named this chapter's subject without describing it: "a DMA write that may be actively mid-burst into host memory this instant, not yet committed in full." This is what that sentence is actually pointing at.

## Overview

Every resource this book has covered so far that carries a "context" -- a Queue Pair, a Memory Region, a Rate Limiter entry -- carries state that outlives any single packet. It's written once, at a specific handoff, and held until something explicitly changes it. This engine has no equivalent. What it tracks is scoped to one transfer: allocated the instant a burst begins, updated continuously while that burst runs, and gone the instant it finishes. Nothing here is installed, and nothing here has a lifecycle worth its own state machine -- a burst is either running or it doesn't exist.

> [!NOTE] Concept
> Read "context" carefully in this Part. Everywhere else in this book, a resource's context is its *identity* -- who it is, what it's negotiated, what it remembers. This engine's in-flight state isn't identity at all. It's scratch space for work already underway, borrowed for the duration of one transfer and handed back the moment that transfer ends.

## Locator

![The master RNIC block diagram with DMA Engine highlighted and every other block dimmed -- this chapter covers what one in-flight transfer's own tracking state contains, not where it points or how many the engine can run at once.](/images/dma/01-in-flight-transfer-state-locator.svg)

The fields below belong to one transfer, for as long as that transfer is running. Where a transfer's addresses actually come from is [Target Selection](/dma/target-selection), immediately next. How many of these can exist on this engine at once, contended across every QP calling into it, is [Multi-QP Burst Arbitration](/dma/multi-qp-burst-arbitration), later in this Part.

## Block diagram

![One in-flight transfer's tracking state: an owning QP number, a direction (gather or scatter), a current address that advances as bytes move, a remaining-length counter that counts down to zero, a scatter-gather cursor naming which segment of a multi-segment transfer is active, and a completion tag linking this burst back to the operation that spawned it.](/images/dma/01-in-flight-transfer-state-block.svg)

## Deep walkthrough

**No standard describes this state, and none should be expected to.** IBTA governs the wire format an operation produces and the semantics software is entitled to assume once it completes -- not the bookkeeping a real engine keeps mid-execution to get there. Every field below is this book's own reasoned inference about what a scatter-gather DMA engine has to track to run correctly, not a structure any specification names or any vendor has published. The same "architect's choice, not a spec requirement" framing this book has applied to QP Table storage and Rate Limiter context applies here too, one level more inevitably: there is no wire-visible or software-visible surface for this state to leak through at all, so no source could confirm a specific shape even if one wanted to check.

**Owning QP.** Every in-flight transfer belongs to exactly one Queue Pair -- the one whose WQE spawned it, on the gather side, or the one whose incoming packet triggered it, on the scatter side. This field is what lets [Single-Burst Fault Handling](/dma/single-burst-fault-handling) attribute a failure to the right connection, and what a completion, once generated, gets tagged with on its way to [CQE Generation](/cq/cqe-generation).

**Direction.** Whether this transfer is *reading* local memory to build an outgoing payload or *writing* incoming data somewhere -- local or remote-exposed, per [Target Selection](/dma/target-selection)'s own resolution. Not a mode switch on the engine as a whole; each in-flight transfer carries its own direction independently, since [Multi-QP Burst Arbitration](/dma/multi-qp-burst-arbitration) allows gather and scatter transfers to be genuinely concurrent, not mutually exclusive phases.

**Current address.** The address the next byte actually moves to or from, right now. This is the one field that changes fastest -- it advances with every beat of the transfer -- and it's also the field [Mid-Transaction Deregistration](/mr/mid-transaction-deregistration) already raised an open question about: whether this address is derived once from a translated, private copy held for the burst's whole duration, or re-derived from the shared Key Cache at some finer granularity. [Mid-Transaction Burst Teardown](/dma/mid-transaction-burst-teardown) picks that question back up rather than resolving it here.

**Remaining length.** A simple countdown -- how many bytes are left to move in this transfer -- that reaches zero exactly when the transfer is done. Paired with current address, this is the minimum state any burst needs to know whether it's finished.

**Scatter-gather cursor.** A WQE's own SGE list, or a large remote-exposed region backed by non-contiguous pages, can describe a transfer as several separate address ranges rather than one contiguous run. This field names which of those ranges is active right now, so the engine knows to advance to the next one rather than declaring the whole transfer complete when the current range's remaining length hits zero.

> [!WARNING] Analogy
> This is the same shape a hardware descriptor-ring walker keeps for a multi-descriptor chain -- current descriptor, current offset within it, descriptors remaining -- applied here to a scatter-gather list instead of a ring. The mechanism of "walk a chain, track where you are in it" isn't RDMA-specific; what's specific is only what the chain describes and where its entries come from, which [Target Selection](/dma/target-selection) covers next.

**Completion tag.** A transfer that finishes has to report back to something -- the operation that requested it, and eventually a Completion Queue Entry. This field is what carries that identity forward through the transfer's execution, so a finished burst can be matched back to the WQE or wire operation it was serving. The actual CQE-generation trigger this feeds is [CQE Generation](/cq/cqe-generation)'s job, not this chapter's -- this field just makes sure the right identity survives long enough to reach it.

**What this state is not.** It carries nothing about permission, bounds, or key validity -- [RX PSN Check and Out-of-Order Arrival](/rx/psn-check-and-out-of-order-arrival) already established that a transfer only ever begins here once an operation has already cleared that checking, on an already-validated address and length. It also carries nothing persistent across transfers: unlike a QP Context field or an MR's registered bounds, none of this survives the burst it belongs to.

## Interface reference

| Field | Carries | Standard's role |
|---|---|---|
| **Owning QP** | which connection this transfer belongs to | architect's choice -- no standard names this state |
| **Direction** | gather (reading local memory) or scatter (writing local or remote-exposed memory) | architect's choice |
| **Current address** | where the next byte moves to or from, right now | architect's choice |
| **Remaining length** | bytes left in this transfer, counting down to zero | architect's choice |
| **Scatter-gather cursor** | which segment of a multi-segment transfer is currently active | architect's choice |
| **Completion tag** | links this transfer back to the WQE or wire operation that spawned it | architect's choice |

## References

- No IBTA or IETF standard governs in-flight DMA tracking state -- it is entirely implementation-specific microarchitecture, with no wire-visible or software-visible surface for any source to describe. Every field above is this book's own reasoned inference from what a scatter-gather transfer mechanically requires to execute and report back, not a citation.
