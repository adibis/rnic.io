---
title: "Eviction Policy"
weight: 8
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "The QP Cache runs out of room the moment enough connections are active at once. Something has to leave to make space for the QP that just arrived, and which QP that is turns out to matter as much as the eviction itself."
prev: /qp/multi-qp-error-arbitration
next: /qp/hardware-assisted-eviction-engines
---

## Locator

![The master RNIC block diagram with the QP Cache highlighted -- this chapter is about which cached QP context gets evicted when that cache is full.](/images/qp/08-eviction-policy-locator.svg)

## Overview

[QP Caching](../qp-caching) established that the on-chip working set has a hard ceiling — around 5,000 QPs on ConnectX-4/5's 2MB of shared SRAM — and that real deployments run past it. When a QP not currently in the cache needs servicing and every slot is occupied, something already resident has to leave first. This chapter is about what decides which one: the policy question, not yet the question of what hardware actually runs that policy, which is the next chapter's subject.

## Block diagram

![The QP Cache full with QP N+1 needing a slot: the Eviction Policy decision box takes recency, priority, and error-state inputs to pick a victim, which is written back to the QP Table if dirty and freed, with QP N+1 fetched into the now-free slot.](/images/qp/08-eviction-policy-block.svg)

## Deep walkthrough

**What a policy has to decide.** Every eviction is a bet: this QP, not any other currently-cached one, is the safest to push back out to the QP Table and re-fetch later if it's needed again. Getting that bet wrong doesn't corrupt anything — the evicted QP's context is still correct in the backing table — but it does mean paying a PCIe-latency refetch (per [QP Caching](../qp-caching)'s numbers) on exactly the QP least able to afford it, which is the failure mode worth designing against.

**LRU (least-recently-used).** Evict whichever cached QP has gone longest without a packet touching it. Cheap to implement, and correct under the assumption this chapter can't actually guarantee: that recent activity predicts near-future activity. It holds well for QPs with sustained, bursty traffic and fails exactly where it matters most — a QP that's quiet for a moment but about to resume a large transfer looks identical, to a pure recency policy, to a QP that's genuinely done. LRU has no notion of *which* QP is expensive to have evicted, only which one happened to be touched least recently.

**Priority-weighted.** Let software (or a control plane above it) mark some QPs as eviction-resistant — a latency-sensitive connection gets weighted to survive cache pressure that would otherwise evict it on pure recency grounds. This fixes LRU's blind spot directly, at a real cost: it's now possible for a small number of high-priority QPs to pin enough of the working set that ordinary QPs thrash against each other for the remaining space, and someone has to own the weights — a policy decision pushed up to software or an operator, not resolved by the cache itself.

**Error-weighted.** Evict cached QPs already showing signs of trouble — elevated retry counts, RNR NAKs, anything the reliability engine (Part 5 (Reliability Engine)) is already tracking per QP — preferentially, on the reasoning that a QP already degrading is one that's cheap to re-fetch relative to the disruption of evicting a healthy one. This is a real, defensible idea, and it comes with a real coupling cost: the caching subsystem now has to consume state from a completely different piece of hardware (the reliability engine's per-QP counters) to make an eviction decision, instead of reasoning purely from its own occupancy and access data. That cross-cutting dependency is exactly the kind of thing that makes a caching structure harder to verify and harder to reason about in isolation, even when the resulting policy behaves well.

None of these three is a universally correct choice, and this book doesn't pick one for the reader. A real design's answer depends on workload assumptions — how skewed the traffic is across QPs, whether latency-sensitive and bulk traffic share one adapter, how much the operator is willing to expose as a tunable versus hard-code. What the next chapter asks is a different question entirely: whichever policy gets chosen, what actually runs it.

No public source documents which of these a real RNIC actually ships, or whether a real implementation blends more than one. That's not a gap in this book's research — vendor cache-replacement policy is not the kind of detail that shows up in a public programming guide. What's above is this book's own reasoned analysis of the tradeoff space a designer actually faces, not a citation.

> **Design rationale.** A pure-LRU cache is the simplest thing that could possibly work, and it's a reasonable default precisely because most real traffic is not adversarial — it's bursty and clustered, and recency really is a decent predictor most of the time. The case for spending real hardware complexity on priority- or error-weighting only holds once a workload actually has QPs whose importance or health genuinely diverges from their recency — a mixed-tenant deployment with SLA-bound and best-effort traffic sharing one adapter, say. Building that complexity in when it isn't needed is pure cost with no payoff; leaving it out when it is needed means the one QP an operator most cares about is exactly as vulnerable to eviction as everything else competing for the same cache line. There's no policy that's free of that trade.

## Interface / timing reference

| Policy | Decision input | Real cost |
|---|---|---|
| LRU | last-access time per cached QPC | blind to which QP is safe vs. costly to evict |
| Priority-weighted | software/operator-set per-QP weight | needs an external owner for the weights; risk of a few QPs pinning most of the cache |
| Error-weighted | live retry/RNR/timeout state from the reliability engine (Part 5) | couples the cache to reliability-engine state, a cross-cutting dependency |

The next chapter asks whether the policy above runs as fixed hardware logic, a programmable engine, or a software-driven trap — a different question from which policy is correct.

## References

- No IETF or IBTA standard governs QP-cache eviction policy -- it is implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above, not a standards document.
