---
title: "Hardware-Assisted Key Cache Eviction"
weight: 8
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "[Queue Pair Management](/qp/) asked what runs an eviction policy: fixed silicon, a programmable engine, or a trap to software. This chapter asks that question once, in one paragraph, and then asks a second one [Queue Pair Management](/qp/) never had to: what runs a flush of the entire cache, and what does that flush cost the lookups it interrupts."
prev: /mr/key-cache-eviction-policy
next: /mr/mid-transaction-deregistration
---

## Overview

[Key Cache Eviction Policy](../key-cache-eviction-policy) named two independent mechanisms living in the same satellite block: ordinary capacity-pressure eviction, which picks one victim translation at a time, and mass invalidation on memory-region deregistration, which clears the working set wholesale with no victim selection at all. This chapter asks what actually runs each one. The first question is identical to one this book has already answered; the second is new.

## Locator

![The master RNIC block diagram with the Key Cache highlighted, the same target as the Key Cache Eviction Policy chapter -- this chapter asks what hardware actually runs each mechanism.](/images/mr/08-hardware-assisted-key-cache-eviction-locator.svg)

## Block diagram

![Two panels: a smaller, desaturated copy of Queue Pair Management's fixed-function/programmable/software-driven eviction-engine spectrum on the left, captioned as identical and not re-derived; on the right, a deregistration call sending a thick global-invalidate arrow straight through the entire Key Cache bar, with several in-flight-lookup tokens along the bar connected by a dashed rail to a red question mark where the arrow crosses it, asking whether the flush waits for them or runs through them.](/images/mr/08-hardware-assisted-key-cache-eviction-block.svg)

## Deep walkthrough

**Capacity-pressure eviction runs on the same spectrum [Queue Pair Management](/qp/) already built.** [Hardware-Assisted Eviction Engines](/qp/hardware-assisted-eviction-engines) worked through the full fixed-function/programmable/software-driven tradeoff for the QP Cache, and because [Key Caching](../key-caching) already established that this is the identical shared SRAM, not a separately-designed structure, that entire analysis carries over to MTT/MPT eviction without modification: the same latency-versus-flexibility spectrum, the same reasoning about why a programmable-but-hardware-resident engine is the realistic middle ground most designs reach for, the same conclusion that a software trap on every miss reintroduces the round-trip RDMA hardware offload exists to remove. [Hardware-Assisted Eviction Engines](/qp/hardware-assisted-eviction-engines) is the one place it's argued, and it applies here exactly as written.

**Mass invalidation is not a smaller version of eviction — it's a different operation entirely.** Ordinary eviction, however it's decided, always has the same shape: pick one occupied slot, write it back if dirty, free it, fill it with the incoming translation. A mass invalidation triggered by a deregistration call has none of that shape. There is no incoming translation waiting for the freed space, no victim to select among candidates, no policy input to consult — the entire cache's occupied MTT/MPT slots (or some large, unpredictable subset of them, per [Key Cache Eviction Policy](../key-cache-eviction-policy)'s finding that a real RNIC is reasoned to flush wholesale rather than sweep out only the deregistered entries) simply stop being valid at once. The hardware mechanism this actually needs is closer to a global clear or a broadcast invalidate signal than to anything resembling a victim-selection engine — cheaper to build, in the sense that it needs no policy logic at all, but not cheaper to execute, for a reason the next paragraph gets into.

**What a flush actually has to coordinate with.** A tag-clear that happens instantaneously, with no regard for what else is touching the cache at that moment, is the simplest possible implementation and also the most dangerous one. [Shared-MR Arbitration](../shared-mr-arbitration) already named the general version of this problem: a lookup already in flight against an entry that a concurrent invalidation is in the process of clearing can, in principle, either complete against a translation that's being retired underneath it or be interrupted partway through. [Mid-Transaction Deregistration](../mid-transaction-deregistration) sharpens that same question specifically for a DMA burst that has already resolved its translation and is actively moving bytes when the flush hits. Neither chapter resolves whether real hardware fences the flush against in-flight lookups (stall new lookups, drain the ones already running, then clear) or lets the flush proceed unconditionally while accepting that a lookup mid-flight might observe a half-torn-down cache — and this chapter doesn't resolve it either; there is no public source describing which choice any real RNIC makes. What this chapter adds is the cost framing: an eviction engine that fences against in-flight work pays that cost once per single-entry eviction, a rare, already-expensive event; an invalidation engine that has to fence the same way pays it across every occupied slot at once, on every deregistration call, which is a categorically larger and more frequent tax if the fencing is real.

**Why this is the one place [Memory Region / Key Table](/mr/)'s hardware question is new.** [Queue Pair Management](/qp/)'s QP Cache never had an operation that invalidates its entire working set as a side effect of one control-plane call on one QP, because tearing down a single QP was never reasoned to touch any other QP's cached context. Memory regions are different — deregistering one is reasoned to cost every other resident translation, unrelated MRs included, exactly the asymmetry [Key Cache Eviction Policy](../key-cache-eviction-policy) already quantified. The eviction-engine question [Queue Pair Management](/qp/) answered stays answered here. The flush-engine question has no [Queue Pair Management](/qp/) analog to point back to.

> [!TIP] Why
> A design that fences the mass-invalidate against every in-flight lookup buys correctness — no lookup ever observes a translation that's simultaneously being torn down — at the cost of turning an unrelated application's deregistration call into a latency event for every other tenant currently touching the cache, however briefly. A design that doesn't fence buys speed and isolation between tenants at the cost of an unresolved race whose actual behavior on real silicon isn't documented anywhere this book found. The asymmetry that makes this a harder call than it looks: an eviction engine's fencing cost is paid by the one workload that caused the pressure, but an invalidation engine's fencing cost, if it exists, is paid by every workload sharing the cache at the moment one of them happens to deregister a region — a blast radius in *time* (stalling unrelated lookups) layered on top of the blast radius in *space* ([Key Cache Eviction Policy](../key-cache-eviction-policy)'s wholesale-flush finding) this part has already established. Neither this chapter nor any source it could find says which tradeoff a real vendor actually made.

> [!IMPORTANT] Food for Thought
> If you had access to a real RNIC and wanted to find out whether deregistration fences against in-flight lookups, what would you actually measure? What experiment would distinguish "the flush waits" from "the flush runs through, and any torn-down read is undefined behavior that just happens not to fail in your test"?

[Mid-Transaction Deregistration](../mid-transaction-deregistration) closes this part by taking the fencing question from a cache-wide concern to a single DMA burst's concern: what happens to one transfer, already in progress, when the translation it depends on disappears out from under it mid-burst.

## References

- No IETF or IBTA standard governs which hardware runs an eviction or invalidation policy -- it is implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above, not a standards document.
