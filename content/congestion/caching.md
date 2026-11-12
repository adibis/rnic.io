---
title: "Caching"
weight: 3
date: 2026-09-12
publishDate: 2026-09-12
draft: false
description: "Every other table this book has covered gets a caching chapter with a citable finding behind it. This one doesn't -- and the honest reason why turns out to be more interesting than a missing source: this resource might be small enough that the question doesn't even apply the way it does to its three siblings."
prev: /congestion/the-rate-limiter-table
next: /congestion/enabling-congestion-control
metaDescription: "Whether an RNIC needs an on-chip cache for per-QP DCQCN rate-limiter state at all, and why the table may be small enough to keep entirely on chip."
seoTitle: "Caching DCQCN Rate-Limiter State"
---

## Overview

[Key Caching](/mr/key-caching) named a real, sourced finding that QP context, memory-region translations, Completion Queue context, and Event Queue context all share one on-chip SRAM pool. No source this book found extends that finding to rate-limiter state. This chapter reasons through whether the same caching relationship -- a small on-chip working set sitting in front of a larger backing store -- actually applies here, and reaches a more qualified answer than any of this book's other caching chapters have needed to give.

> [!CAUTION] Gotcha
> This chapter is not a fourth confirmed instance of Key Caching's finding. Read every claim below as this book's own reasoning, clearly labeled where it is, not as an extension of a citation that doesn't cover this resource.

## Locator

![The master RNIC block diagram with Rate Limiter highlighted and every other block dimmed -- this chapter asks whether an on-chip working set sits in front of this structure the way one does for its three table-shaped siblings.](/images/congestion/03-caching-locator.svg)

[The Rate Limiter Table](/congestion/the-rate-limiter-table) covered the storage structure and its addressing. This chapter is about whether that structure needs a cache in front of it at all.

## Block diagram

![Two candidate shapes side by side: on the left, the familiar hit/miss pattern QP Cache and Key Cache both use, a small on-chip working set backed by a larger DRAM-resident table; on the right, a single on-chip structure holding every QP's rate-limiter entry at once, no miss path because there's nothing to miss. A back-of-envelope size estimate sits between them -- five small fields per QP, plausibly cheap enough in aggregate to make the right-hand shape realistic at QP counts this book has already discussed.](/images/congestion/03-caching-block.svg)

## Deep walkthrough

**Why this question even needs asking, when the answer to "does congestion control need caching" might just be no.** [QP Context](/qp/qp-context) is rich enough, and [Key Caching](/mr/key-caching)'s memory-translation entries numerous enough, that neither could plausibly stay resident on-chip for every QP or every registered region an adapter supports -- that's exactly the gap a working-set cache exists to bound. [Rate-Limiter Context](/congestion/rate-limiter-context) is a different shape: five small fields -- two rates, one reaction factor, and a timer and a byte counter that each carry their own stage count -- per QP, not per operation and not per registered memory region. A rough estimate, not a sourced figure: a few dozen bytes per QP, which stays in the low single-digit megabytes even at QP counts in the hundreds of thousands. Whether that's small enough for a real implementation to keep entirely on-chip, for every QP it supports, with no working-set/backing-store split at all, is a real open question this chapter can reason about but not settle.

> [!TIP] Why
> A resource this small changes the caching tradeoff, if not the underlying instinct. [QP Caching](/qp/qp-caching)'s own Why callout already named the real tension: how much scarce on-chip SRAM to spend on one structure versus every other structure competing for the same die area. That tension is sharpest when a resource is large enough that "cache everything" is obviously wasteful. Five small fields per QP may simply not be large enough to make that tradeoff bite the same way -- which would mean the honest answer to "what's this resource's eviction policy" is closer to *there may not be one*, not because this book is declining to look, but because the resource this Part has actually described may not need one.

**If it does need a working-set/backing-store split, the shape is the familiar one.** Should a real implementation's rate-limiter entry turn out to be larger than this chapter's estimate, or should on-chip SRAM budget be tighter than assumed here, the caching relationship would look exactly like [QP Caching](/qp/qp-caching)'s and [Key Caching](/mr/key-caching)'s: a small on-chip working set of actively-touched QPs' rate-limiter entries, backed by [The Rate Limiter Table](/congestion/the-rate-limiter-table)'s full, DRAM-resident population, with a miss costing the same PCIe round trip either of those two chapters already describe.

> [!WARNING] Analogy
> This is the same "maybe it just fits" question a CPU's own architects face with small, per-thread control state -- a handful of control registers might live in dedicated on-die storage for every hardware context a core supports, with no working-set question at all, while a much larger per-thread structure (a TLB's own translation entries, a branch predictor's history tables) has no choice but to be a bounded cache. Size, not category, is what decides which shape a given piece of state actually needs.

**Where this connects.** If a working set does exist, [Eviction Policy](/congestion/eviction-policy), next, covers what happens when it's full. If it doesn't -- if every QP's rate-limiter entry really does stay resident on-chip for that QP's whole life -- then eviction, in the cache-miss sense this book has used everywhere else, has no subject to describe here at all, and that chapter's job becomes naming why, rather than picking a policy.

> [!IMPORTANT] Food for Thought
> This chapter reasoned from a rough size estimate, not a measurement of any real chip. What would actually settle this question -- and would you expect the answer to be the same for a NIC supporting a few thousand QPs as for one supporting hundreds of thousands?

## References

- No source this book found describes Rate Limiter state's on-chip working-set behavior, size, or eviction model directly -- unlike [Key Caching](/mr/key-caching)'s QPC/MTT-MPT/CQC/EQC finding, which explicitly does not extend to this resource. Every claim above is this book's own reasoned estimate, flagged as such throughout, not a citation.
- Zhu, Eran, Firestone, Guo, Lipshteyn, Liron, Padhye, Raindel, Haj Yahia, Zhang, "Congestion Control for Large-Scale RDMA Deployments," SIGCOMM 2015 -- the field list this chapter's size estimate is built from.
