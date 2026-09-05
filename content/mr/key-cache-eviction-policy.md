---
title: "Key Cache Eviction Policy"
weight: 7
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "The QP Cache only ever loses one entry at a time to eviction pressure. The Key Cache carries that same policy question, plus a second and harsher one: a single deregistration call is reasoned to invalidate the whole cache at once, not just the key being retired."
prev: /mr/shared-mr-arbitration
next: /mr/hardware-assisted-key-cache-eviction
---

## Locator

![The master RNIC block diagram with MR/Key Table and Key Cache highlighted -- this chapter covers what decides which cached translation gets evicted, and the separate mass-invalidation trigger deregistration causes.](/images/mr/07-key-cache-eviction-policy-locator.svg)

## Overview

[Key Caching](../key-caching) established that the Key Cache is not a dedicated structure — it is the same on-chip SRAM [Part 1](/qp/) (Queue Pair Management)'s QP Cache lives in, contended across QP context, MTT/MPT translations, completion-queue, and event-queue state alike, with a real, measured miss cost when a translation isn't resident (roughly 2.6 μs to 5.1 μs round-trip once the working set overflows the cache). [Shared-MR Arbitration](../shared-mr-arbitration) covered what happens when multiple QPs reference one live entry at the same time. This chapter asks what happens when an entry has to leave that cache — and unlike Part 1's QP Cache, this question has two genuinely different answers depending on what triggered the departure: ordinary capacity pressure, which needs a policy to pick a victim, and memory-region deregistration, which is reasoned to clear out the whole cache at once and needs no victim-selection at all.

## Block diagram

<!-- IMAGE: Zoomed diagram of the Key Cache, split into two side-by-side labeled panels. Left panel, "Capacity pressure": the cache shown with every slot occupied by a small "MTT/MPT" box; an incoming arrow labeled "new translation needed, cache full" points at the cache from the left; a small decision box labeled "Eviction Policy" sits between the cache and one highlighted victim slot (drawn in the red fault/emphasis color per docs/SVG_STYLE.md, distinct from the neutral occupied slots), fed by three small labeled input arrows: "recency," "priority," "error state" (the last dashed, crossing in from the reliability engine, mirroring /qp/eviction-policy's diagram exactly); an arrow from the victim slot down to "MR/Key Table (DRAM/HBM)" labeled "written back if dirty, then freed." Right panel, "Deregistration": a sky-blue actor box labeled "ibv_dereg_mr()" (per SVG_STYLE.md's actor-initiating-a-request color) sends one arrow labeled "deregister lkey/rkey N" that does not point at a single slot but fans out across the entire Key Cache boundary, drawn with a thick red outline enclosing every occupied slot at once, labeled "entire cache invalidated, not just entry N" -- the single red slot in the left panel contrasted against the all-slots red enclosure in the right panel to make the difference in blast radius legible without reading either caption. -->

## Deep walkthrough

**The ordinary case: picking a victim.** When the cache is full and a translation not currently resident needs servicing, something already cached has to leave first, and [Eviction Policy](/qp/eviction-policy) already worked through the three shapes that decision can take: LRU (evict whichever cached translation has gone longest untouched), priority-weighted (let software pin translations belonging to latency-sensitive connections against eviction), and error-weighted (evict translations tied to MRs or QPs already showing trouble first, on the theory that they're cheaper to re-fetch than to keep). Every word of that chapter's reasoning about the tradeoff — recency as an imperfect predictor, priority weights needing an external owner, error-weighting coupling the cache to state it doesn't otherwise track — applies to MTT/MPT entries without modification. This book won't re-derive it a second time; the mechanism doesn't change just because the cached object is a translation instead of a QP context.

**A second, unrelated reason an entry leaves.** What Part 1's QP Cache never had to reckon with is a single control-plane call that empties out a large share of the cache regardless of recency, priority, or error state. Kong et al. (NSDI 2023), measured directly against an NVIDIA ConnectX-5, give this a real number: a victim connection running alone sustains 96.6 Gbps at a 17.2% MTT cache miss rate. A concurrent attacker that only *registers* new MRs (roughly 5,000/sec) barely moves that needle — 95.9 Gbps, 22.9% miss rate. A concurrent attacker that *deregisters* MRs instead collapses the same victim to 48.0 Gbps at a 49.1% miss rate — the victim's bandwidth roughly halves. The paper's own explanation is the load-bearing fact for this chapter: MR deregistration is reasoned to invalidate the entire MTT/MPT cache wholesale, not just the one entry being torn down, apparently to avoid the possibility of a stale MR reference resolving against freed or repurposed memory.

That is a different kind of event from anything section one describes, not a more extreme version of it. Capacity eviction always has a designated replacement — the incoming reference that needed the freed slot — and it only ever removes the one entry the policy selected. Deregistration's invalidation has no such symmetry: it can be triggered by tearing down a single MR, and its cost lands on every other translation that happened to be resident at the time, including ones with no relationship whatsoever to the MR being deregistered. A workload with dozens of live registrations loses all of them to one process's teardown of one, and every one of those translations has to be paid for again through the same PCIe round-trip [Key Caching](../key-caching) already charges for a cold miss — not a transient blip, but a sustained recovery cost across however many MRs were resident when the flush hit.

**Why wholesale rather than targeted.** No public source documents the actual invalidation mechanics — this is reasoned analysis, not a described implementation. But a plausible reason a real cache would flush everything rather than sweep out only the deregistered key's entries: doing the targeted version correctly requires the cache to find every line associated with that lkey/rkey, which may span many MTT entries if the registration covered more than a few pages, and finding all of them with certainty is a harder correctness problem than clearing the array and letting everything re-warm. When the cost of getting a targeted invalidation wrong is a stale translation continuing to resolve against memory the kernel may have already unmapped, a designer has a real incentive to prefer a mechanism that is trivially, provably complete over one that is merely fast.

**What this means for "the" eviction policy.** There is no longer a single eviction policy for this cache — there are two independent mechanisms answering two independent questions. Whichever of LRU, priority-weighted, or error-weighted policy governs ordinary capacity pressure has no say whatsoever over the deregistration case: mass invalidation doesn't consult recency, priority, or error state, because it isn't choosing among candidates. It's clearing the field. Naming both explicitly, as this chapter does, is worth doing precisely because they're easy to conflate under one label ("eviction") when they are, in the failure mode that matters, unrelated events with unrelated remedies.

Beyond the Kong et al. finding, this chapter shares Part 1 chapter 8's epistemic status: no public source describes which of the three ordinary-case policies a real RNIC's key cache actually implements, or whether any implementation blends more than one. `[SPARSE PUBLIC SOURCING]`. What's different from Part 1 is that this chapter isn't thin everywhere — the mass-invalidation trigger is a real, primary-source-verified data point this book can build the chapter's center of gravity around, rather than a footnote appended to unsourced reasoning.

> **Design rationale.** A targeted invalidation that has any chance of missing a line is a worse failure than an untargeted one that's merely more expensive. If the thing being protected against is a remote peer resolving a stale rkey against memory the OS has already reassigned, "clear everything, cheaply and provably" beats "clear precisely, and hope the sweep found every line" — the cost of the conservative choice is paid in refetch latency, an inconvenience Key Caching already quantifies; the cost of the aggressive choice, gotten wrong, is a correctness violation. That asymmetry is exactly why a mass-invalidation trigger belongs in the same chapter as ordinary eviction policy rather than off in a footnote: it's not a more extreme eviction, it's a different kind of guarantee, purchased at a different kind of price.

## Interface / timing reference

| Trigger | Decision input | Real cost |
|---|---|---|
| Capacity pressure — LRU | last-access time per cached translation | blind to which translation is safe vs. costly to evict, same tradeoff as the QP Cache |
| Capacity pressure — priority-weighted | software/operator-set weight per MR or connection | needs an external owner for the weights; risk of a few MRs pinning most of the cache |
| Capacity pressure — error-weighted | live error/violation history from the reliability path or permission-violation counters | couples the key cache to state it doesn't otherwise track |
| Deregistration — mass invalidation | none — the dereg call itself is the only input; no victim is selected | wholesale flush of the working set, measured to roughly halve victim throughput on at least one real RNIC (Kong et al., ConnectX-5) regardless of which capacity policy governs the ordinary case |

The next chapter turns to what hardware actually runs both of these mechanisms, and asks whether the mass-invalidation trigger just described changes where the answer should sit on the fixed-function/programmable/software-driven spectrum [Hardware-Assisted Key Cache Eviction](../hardware-assisted-key-cache-eviction) opens with.

## References

- No IETF or IBTA standard governs key-cache eviction policy and mass-invalidation mechanics -- it is implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above, not a standards document.
