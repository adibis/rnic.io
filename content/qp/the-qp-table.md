---
title: "The QP Table"
weight: 2
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "A QP number isn't a name, it's an address. This chapter is about the structure that address indexes into, and why a page-table-shaped answer fits a problem the public record barely documents."
prev: /qp/qp-context
next: /qp/qp-caching
---

## Locator

![The master RNIC block diagram with the QP Table highlighted and every other block, including QP Cache, dimmed -- this chapter is the backing store one QP Context entry lives in.](/images/qp/02-the-qp-table-locator.svg)

[QP Context](/qp/qp-context) covered what's inside one entry. This chapter is about the structure holding every entry, and how a QP number finds the right one.

## Overview

Every QP context described in the previous chapter lives somewhere: one entry in a table sized to hold as many QPs as the adapter supports, addressed by the QP number that came back when the QP was created. This chapter covers that table's shape -- how a QP number becomes an index, and why an indexed, page-table-like structure is the natural answer to a problem this book can otherwise document only thinly.

## Block diagram

<!-- IMAGE: A single large box labeled "QP Table (DRAM/HBM-backed)", subdivided into a column of smaller row-boxes each labeled "QPC" with a QP number to its left (e.g. 0x0001, 0x0002, ... shown as a handful of rows with an ellipsis implying many more). An arrow from outside labeled "QP number (from BTH dest QP field on RX, or the QP being scheduled on TX)" pointing at the table with a magnifying-glass icon over one specific row, captioned "direct or shallow multi-level index -- not a search." A second, smaller box drawn overlapping the top of the table, dashed outline, labeled "QP Cache (chapter 1.3) -- the working set actually consulted first," with an arrow from the main lookup path branching to check this box before falling through to the full table. Below the main table, a second, smaller inset panel titled "Two shapes for the same table" -- left side, "Flat table": one contiguous box sized to the maximum supported QP count, every slot pre-allocated whether a QP occupies it or not, captioned "cheapest lookup, wasted memory on unused range." Right side, "Shallow multi-level": a small top-level directory box with a few populated pointers fanning out to only the second-level page blocks that actually have QPs in them, gaps left unallocated, captioned "one extra pointer dereference (cache-backed), no memory spent on QP numbers nobody's using." -->

This chapter's table diagram now carries two panels: the lookup path itself, and a second inset making the design-rationale sidebar's flat-vs-multi-level tradeoff visible rather than leaving it only in prose.

## Deep walkthrough

**What the index actually is.** A QP number is assigned once, when the QP is created, and held fixed for that QP's entire life -- it never gets reassigned to a different table entry while the QP exists. That's what makes a QP number usable as a direct table index rather than a key that needs resolving: the adapter (or the driver, coordinating with the adapter's own resource allocator) hands out QP numbers from a bounded range it manages itself, which means the numbers in active use are dense or near-dense within that range, not sparse values scattered across a huge address space. A dense, hardware-assigned index is exactly the condition that makes a flat, directly-indexed table -- or, once the range gets large enough that a single flat table would waste too much memory on unused entries, a shallow multi-level table walked the same way a page table is -- the obvious structural fit. This is the "page-table analogy" in its most literal form: not that a QP table walk resembles a virtual-memory walk in its mechanics, but that both solve the same problem, a huge notional address space with a much smaller dense working subset, with the same kind of answer.

**Where this gets checked.** Two places ask this table the same question, from opposite ends of a connection: the TX side, resolving the QP number of whichever QP is about to be scheduled, and the RX side, resolving the destination QP number just read out of an arriving packet's BTH ([What Is RoCEv2](/overview/what-is-rocev2)). Neither lookup goes directly to this table in the common case -- [QP Caching](/qp/qp-caching), next, is the structure actually consulted first, with this table as the backing store a cache miss falls through to.

**When a lookup doesn't resolve.** A QP number that doesn't correspond to any currently-allocated QP -- one that was never assigned, or one that belonged to a QP already destroyed -- has to fail cleanly rather than resolve to a stale or garbage entry. The public record doesn't document the specific mechanism a real RNIC uses to distinguish "this index is currently valid" from "this index is unallocated" (a valid bit per entry, a separate allocation bitmap, or something else) -- treat this as an open implementation detail rather than a settled fact. What's certain is that the failure has to be cheap and immediate: a packet naming an unallocated destination QP is a malformed or stale packet, not a condition worth a slow-path investigation.

**What happens after a QP number is freed.** This chapter has so far only discussed a QP number *while the QP exists* -- assigned once, held fixed for that QP's life. It says nothing yet about what happens to that number once the QP is destroyed. A bounded range that's dense with QPs in active use is also, eventually, a range whose freed numbers get handed out again to newly-created QPs -- and a stale, delayed, or maliciously-replayed packet naming an old QP number that has since been reallocated to an unrelated new QP is a real correctness hazard, structurally the same shape as a stale TLB entry or an ABA problem: the index resolves to *something*, just not the something the packet's sender actually meant. Whether a real implementation quarantines a freed QP number for some interval before reissuing it, or reuses it immediately and relies entirely on other mechanisms (PSN state, connection-level authentication) to reject a stale packet that happens to land during the gap, is not documented anywhere this book found -- flagged here as an open question, in the same spirit [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) treats the state that survives a reset.

**What happens when the table itself is full.** Everything [QP Caching](/qp/qp-caching) covers next is about cache pressure -- a QP that exists in this table but isn't currently resident in the on-chip working set. That's a different failure mode from this one: what happens when a QP *creation* request arrives and this table has no free entry left to allocate at all, because the adapter has hit its maximum supported QP count. This isn't a cache-eviction question -- there's no cached copy to evict, no working set to shrink, just a hard ceiling on how many QPs can exist at once. The verbs-layer consequence is a failed allocation call, returned to software synchronously, rather than anything that shows up later on the wire -- closer in shape to any resource-allocation failure (out of file descriptors, out of physical memory) than to the traffic-driven failures the rest of this Part spends most of its time on.

**What this table is not.** It's worth being precise about scope here, because it's easy to over-load this chapter with material that belongs elsewhere: this table's job is holding QP context, addressed by QP number. It has nothing to do with translating a virtual address to a physical one -- that's [Part 2](/mr/) (Memory Region / Key Table)'s Key Table, keyed by `lkey`/`rkey` instead, a structurally similar idea applied to a completely different resource.

> **Design rationale.** The honest state of public documentation here is thin: no source found describes a specific modern RNIC's QP table organization in the kind of detail this book would need to state as settled fact. What follows is reasoned architectural inference, not a citation. A flat, directly-indexed table is the cheapest possible structure to build and to look up -- one multiply-and-add from QP number to address, no walk, no intermediate levels -- and it's a perfectly reasonable design as long as the supported QP count stays small enough that a fully-populated table is affordable in DRAM/HBM regardless of how many QPs are actually in use at once. Once the supported QP count grows large enough that provisioning a flat table sized to the maximum, rather than the typical, deployment becomes wasteful, a shallow multi-level structure -- allocate second-level table pages lazily, only for QP-number ranges that actually have QPs in them -- trades one extra pointer dereference on the (cache-backed, so usually cheap) miss path for real memory savings on hardware that has to support a QP count far larger than any single deployment will actually use concurrently. Which side of that tradeoff a real chip lands on is a capacity-planning decision this book can reason about but not report on with citations.

## Interface reference

| Property | Answer |
|---|---|
| Addressed by | QP number (assigned once, at QP creation, fixed for the QP's life) |
| Backing store | DRAM or on-package HBM -- off-chip, not on-chip SRAM |
| Addressing scheme | Direct index (flat table) or shallow multi-level index (page-table-style), depending on supported QP-count range -- implementation-specific, not standardized |
| Entry contents | One QP Context ([chapter 1.1](/qp/qp-context)) per entry |
| What's consulted first on a real lookup | The QP Cache ([chapter 1.3](/qp/qp-caching)) -- this table is the miss path, not the common path |
| Invalid-index handling | Must fail cleanly and cheaply; specific mechanism not publicly documented |
| Freed QP number reuse | Quarantine before reissue vs. immediate reuse -- not publicly documented |
| Table-full behavior | QP creation fails synchronously in software; no cache-eviction question involved (contrast with [QP Caching](/qp/qp-caching)) |

[QP Caching](/qp/qp-caching) is next: the on-chip working set that makes touching this table on every packet unnecessary, and what a cache miss against it actually costs.

## References

- No IETF or IBTA standard governs QP table/cache storage and addressing -- it is implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above, not a standards document.
