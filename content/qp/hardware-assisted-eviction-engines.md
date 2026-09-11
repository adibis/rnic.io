---
title: "Hardware-Assisted Eviction Engines"
weight: 9
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "Deciding which QP to evict is one question. Deciding what actually runs that decision -- fixed silicon, a programmable engine, or a trap to software -- is a separate one, and it's where the cost of flexibility gets paid in latency, not complexity."
prev: /qp/eviction-policy
next: /qp/mid-transaction-qp-reset
---

## Overview

[Eviction Policy](../eviction-policy) asked which cached QP should be evicted under pressure. This chapter asks a separate question: what actually executes that decision, on every single cache miss, without becoming the bottleneck the cache itself exists to avoid. The answer sits somewhere on a spectrum from fully fixed hardware to a full software trap, and where a real design lands on that spectrum is an engineering trade, not a solved problem.

## Locator

![The master RNIC block diagram with the QP Cache highlighted, the same target as the Eviction Policy chapter -- this chapter asks what hardware actually runs that policy.](/images/qp/09-hardware-assisted-eviction-engines-locator.svg)

## Block diagram

![A horizontal spectrum from fixed-function hardware through a programmable engine to a software-driven trap that leaves the chip entirely, with latency and flexibility bars both growing longer left to right.](/images/qp/09-hardware-assisted-eviction-engines-block.svg)

## Deep walkthrough

**Fully fixed-function hardware.** The eviction policy from the previous chapter is wired directly into the cache controller's logic at design time — an LRU bit per line, say, updated and consulted entirely in hardware, with no register interface exposed to change how it behaves. This is the fastest possible answer for the decision itself: a cache miss resolves *which* line to evict in the same cycle-scale path as the rest of the lookup, with nothing waiting on anything off-chip. What that decision triggers next -- writing the victim's context back to the QP Table, a real cost for any QP that has processed even one packet since being cached, per [Eviction Policy](../eviction-policy) -- is a separate, inherently off-chip operation no amount of decision-speed changes; fixed-function hardware makes picking the victim fast, not evicting it free. The cost is inflexibility baked in for the life of the silicon — if a later workload needs error-weighted eviction and the chip only implements LRU, there is no field upgrade path; the next design has to change it.

> [!WARNING] Analogy
> A CPU faces the identical spectrum for its own cache replacement policy, and real silicon occupies every point on it. Most L1/L2 caches hardwire a pseudo-LRU tree in logic, exactly the fixed-function end of this spectrum. Intel's Cache Allocation Technology sits in the middle: still hardware-resident and still cycle-fast, but exposed through model-specific registers a driver can tune to change how ways get partitioned between workloads, without touching the replacement logic itself. Full software-driven replacement barely exists in a CPU's own caches, for the same reason this chapter argues it's a hard sell here -- but an OS's virtual-memory page replacement is exactly that pattern, one level up the memory hierarchy, where a much larger miss penalty (a page fault, not a cache line refill) actually justifies paying a software round trip.

**A programmable hardware engine.** Keep the decision in hardware, but expose it through configuration: weight registers software can tune, a small threshold table, or a compact embedded sequencer running a constrained policy program, none of which touch the fast path's cycle budget on a hit. This is the realistic middle ground most serious designs actually reach for — it keeps eviction decisions off the software critical path entirely (a miss still resolves without leaving the chip to ask permission), while giving an operator or driver a real lever to shift behavior between, say, pure LRU and priority-aware eviction, without a silicon respin.

What "doesn't touch the fast path's cycle budget" actually requires depends on which of those three shapes is doing the work. A weight-register comparison or a small threshold-table lookup is combinational or close to it -- realistically no slower than the fixed-function LRU-bit read it's replacing, since both are small, bounded lookups gated by the same miss signal. A microcoded sequencer is a different case: even fully on-chip, walking a constrained instruction stream to score candidates takes real cycles, likely a small, fixed number set by the sequencer's own instruction-count ceiling rather than anything data-dependent -- a design that let the program's running time vary with input would have reintroduced exactly the unpredictable-latency problem fixed-function hardware exists to avoid. Whichever shape a real design picks, "programmable" and "cycle-fast" aren't in tension; "programmable" and "unbounded" would be, and no serious design would ship the second one.

> [!CAUTION] Gotcha
> "Programmable" doesn't mean "free of the cost this chapter is measuring" -- it means the cost gets paid in a fixed, bounded number of extra cycles instead of an open-ended one. A sequencer whose running time could vary with which candidates it happens to be scoring has quietly become a smaller version of the software-driven problem this chapter warns against, just moved on-chip instead of off it. The thing that actually keeps a programmable engine in the "realistic middle ground" is a hard ceiling on its own execution time, not merely the fact that it never leaves the chip.

**A technique that blurs the line between these further: deciding the victim before it's needed.** Because whatever policy state a design uses -- LRU recency, a priority weight, an error flag -- already updates on every cache access regardless of whether a miss is imminent, a real implementation doesn't have to wait for a miss to start picking a victim. It can maintain a standing "if a line has to go, this is the one" answer continuously in the background, updated incrementally alongside normal hits, so that by the time a miss actually happens the victim is already known and only the eviction itself -- the write-back and refill -- remains on the critical path. Done this way, even a policy complex enough to nominally need several cycles of evaluation can present as effectively free at miss time, because none of that evaluation happened on the miss's own clock. This isn't a fourth point on the spectrum; it's an implementation technique that can apply to the fixed-function or programmable ends of it, and it's exactly the kind of optimization a public source is unlikely to ever confirm for any specific RNIC -- stated here as a plausible, well-precedented technique, not a documented one.

![Two timelines. Without precomputation, a miss is followed by "decide victim" sitting squarely on the critical path before eviction can proceed. With precomputation, ordinary hits continuously update a standing victim candidate in the background, so a miss goes straight to eviction with the victim already known.](/images/qp/09b-hardware-assisted-eviction-precompute.svg)

**Software-driven.** Hardware detects cache pressure and traps: "eviction needed, pick a victim," handed off to firmware or a driver running on a management core. This is maximally flexible — arbitrary policy logic, easy to update, easy to instrument — and it reintroduces exactly the cost this book's opening chapter ([What Is RDMA](/overview/what-is-rdma)) named as the thing hardware offload was supposed to remove: a software round trip sitting on a path that used to be, or was supposed to be, hardware-only. A cache miss is already the expensive case relative to a hit; routing its resolution through software makes the expensive case measurably more expensive, on top of the PCIe fetch [QP Caching](../qp-caching) already charges it.

None of the three is the universally correct answer, and no public source documents which real RNICs land where on this spectrum for QP-context eviction specifically — this is squarely inside vendor silicon that isn't publicly described. What follows is this book's own reasoned analysis of what pushes a design toward one end or the other, not a report of any one chip's actual implementation.

The workload shape drives the choice. A deployment where eviction pressure is rare and traffic is fairly uniform can tolerate fixed-function hardware's inflexibility, because the policy question barely matters in practice. A deployment running mixed-priority tenants on shared adapters, where 1.8's priority- or error-weighting changes outcomes materially, needs at least a programmable engine to make that weighting adjustable without a new chip. A software-driven trap only earns its cost where eviction events are rare enough, or policy sophistication valuable enough, that occasional software latency on a cache miss is a fair trade against the flexibility gained — plausible for a research or highly specialized platform, a hard sell for a general-purpose line-rate NIC.

> [!TIP] Why
> The instinct to reach for maximum flexibility — trap to software, let firmware decide anything — collides directly with the reason a QP cache exists at all: [QP Caching](../qp-caching) built this structure specifically to keep per-packet hot-path work off of PCIe and off of software, because that round trip is exactly what a cache miss already costs. Handing the *eviction decision itself* back to software on every miss doesn't eliminate that cost, it just relocates and probably compounds it. A programmable-but-hardware-resident engine is less a compromise than the design that actually respects the reason the cache was built in the first place: keep every step of a fast-path miss in hardware, and spend the flexibility budget on registers and thresholds a driver can tune, not on a round trip that leaves the chip.

> [!IMPORTANT] Food for Thought
> A shared adapter runs a stable, well-behaved tenant mix 99% of the time, with occasional bursts of mixed-priority contention from a handful of noisy tenants. Which point on this spectrum would you actually build, and would you build it differently if the bursts were rare but severe versus frequent but mild? What telemetry would you want from a fixed-function design before committing silicon budget to a programmable one?

That closes out the caching-and-eviction half of this part. The last chapter turns to a different kind of hazard entirely: what happens when a QP doesn't just get quietly evicted, but gets reset while an operation against it is still in flight.

## References

- No IETF or IBTA standard governs which hardware runs an eviction policy -- it is implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above, not a standards document.
