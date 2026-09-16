---
title: "Address Translation Handoff"
weight: 3
date: 2026-09-16
publishDate: 2026-09-16
draft: false
description: "This chapter is short on purpose. Every hard question about how a key becomes a physical address already has an owner two Parts back -- this chapter's only job is naming exactly where in this engine's own flow that lookup gets called, and what this engine is and isn't allowed to assume about its answer."
prev: /dma/target-selection
next: /dma/the-gather-path
---

## Overview

Neither address [Target Selection](/dma/target-selection) names is usable yet -- a software-named `lkey` and a wire-named `rkey` are both still keys, not physical addresses. Turning one into the other is [Key Caching](/mr/key-caching)'s mechanism in full, already covered field by field, hit and miss, eviction and all. This chapter doesn't re-derive any of that. It names one thing: this is the stage that calls it.

> [!NOTE] Concept
> "Address Translation" isn't this engine's own lookup structure -- it's this engine's name for the moment it hands a key to [Key Caching](/mr/key-caching) and waits for a physical address back. The structure, the working set, and the eviction policy behind that answer all belong to [Memory Region / Key Table](/mr/), not here.

## Locator

![The master RNIC block diagram with DMA Engine and Key Cache both highlighted -- this chapter is the one place this Part's own flow reaches directly into Part 2's territory.](/images/dma/03-address-translation-handoff-locator.svg)

## Block diagram

![A key -- lkey or rkey, from Target Selection -- entering Key Cache; a hit returning a physical address and the permission/bounds verdict together, in one combined lookup; a miss dropping to the backing Key Table with the same eventual answer, per Key Caching's own hit/miss mechanics, referenced rather than redrawn.](/images/dma/03-address-translation-handoff-block.svg)

## Deep walkthrough

**One combined lookup, not two sequential ones.** [Permission and Bounds Violations](/mr/permission-and-bounds-violations) already stated this plainly for the check itself: no source documents bounds and permission validation as separate stages run in a specific order, or even confirms they're separate at all rather than "one combined table-entry validation that either passes or doesn't." This chapter treats address resolution the same honest way -- the single call into Key Cache this stage makes is what produces a physical address *and* a pass/fail verdict together, not two lookups with one running first.

**What this engine hands over, and what it gets back.** A key (`lkey` or `rkey`, per Target Selection), the address and length the operation names, and which kind of access this transfer needs (read, for a gather; write, for a scatter) go in. A physical address comes back on a pass; a fail comes back as the same rejected-lookup outcome [Permission and Bounds Violations](/mr/permission-and-bounds-violations) already covers in full, including its own honest admission that hardware can't distinguish a bounds failure from a permission failure in what it reports. This chapter adds nothing to that reporting -- a failed lookup here is that chapter's event, not a new one.

**Reading [RX PSN Check and Out-of-Order Arrival](/rx/psn-check-and-out-of-order-arrival)'s handoff precisely.** That chapter describes an arriving packet as already having "cleared permission and bounds checking" by the time it reaches this engine. Read that as naming the *complete set of conditions* a payload needs satisfied before it can land, not as asserting a separate, earlier pipeline stage that ran the check before this one exists. The lookup that actually clears permission and bounds is this stage's own call into Key Cache -- the same lookup, not a redundant second one.

> [!CAUTION] Gotcha
> Don't go looking for a distinct "permission pre-check" stage somewhere in RX Pipeline that runs before this one. Nothing this book has found describes one, and Permission and Bounds Violations' own position -- one combined validation, no documented internal order -- argues against inventing one here for tidiness.

**This engine is a reader here, and nowhere else either.** [MR Capacity and Timing](/mr/mr-capacity-and-timing) already established this fact directly: "nothing about a routine DMA writes back into the address, length, permission, or translation fields it consulted." Every lookup this stage makes is read-only against Key Cache -- registration and deregistration are the only two events that ever write that structure, both arriving over a control path this engine never touches.

**Where this connects.** A passing lookup hands a physical address forward into [The Gather Path](/dma/the-gather-path) or [The Scatter Path](/dma/the-scatter-path), whichever Target Selection's direction called for. Whether this engine holds that resolved address privately for a whole burst, or re-derives it from Key Cache more than once during a long transfer, is the open question [Mid-Transaction Burst Teardown](/dma/mid-transaction-burst-teardown) covers, not this chapter -- this chapter only names the single call that happens at least once, at the start.

## Interface reference

| Question | Answer |
|---|---|
| **What performs the lookup** | [Key Caching](/mr/key-caching) -- not a structure this engine owns |
| **What goes in** | a key (`lkey`/`rkey`), the named address and length, and the access type (read or write) |
| **What comes back on a pass** | a physical address |
| **What comes back on a fail** | the same rejected-lookup event [Permission and Bounds Violations](/mr/permission-and-bounds-violations) covers -- no separate DMA-Engine-specific fault type |
| **Is this a read or a write against Key Cache** | always a read -- confirmed directly by [MR Capacity and Timing](/mr/mr-capacity-and-timing) |

## References

- No IBTA standard names an "address translation handoff" as its own mechanism -- this chapter is entirely a pointer to [Memory Region / Key Table](/mr/)'s own, already-cited research, applied here only to name the calling convention.
