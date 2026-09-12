---
title: "Multi-Flow CNP Arbitration"
weight: 8
date: 2026-09-12
publishDate: 2026-09-12
draft: false
description: "Every other multi-instance chapter in this book is about many QPs contending for one shared, serialized gate on the send side. This Part doesn't have one of those -- each QP's rate limiter runs entirely independent state. The real contention here is on the other side of the connection entirely, and it's a bounded, measured, real hardware number, not a hypothetical."
prev: /congestion/single-qp-rate-state-faults
next: /congestion/eviction-policy
---

## Overview

[Multi-QP Error Arbitration](/qp/multi-qp-error-arbitration), [Event and Multi-CQ Arbitration](/cq/event-and-multi-cq-arbitration), and [Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling) each asked the same question of a different shared, serialized send-side resource. This chapter asks it too, and the honest answer is that this Part doesn't have one of those to arbitrate at all -- what it has instead is a bounded resource on the *receive* side, with a real, measured capacity this book can cite directly rather than reason toward.

## Locator

![The master RNIC block diagram with CNP Generation highlighted and every other block dimmed -- this chapter is about what happens when more flows need a CNP at once than that structure can generate.](/images/congestion/08-multi-flow-cnp-arbitration-locator.svg)

[Single-QP Rate-State Faults](/congestion/single-qp-rate-state-faults) covered one QP's own failure conditions. This chapter is about many flows needing the same structure at once -- not on the send side, where every other multi-instance chapter in this book has looked, but on the receive side, where [CNP Generation](/congestion/cnp-generation) actually lives.

## Block diagram

![Two shapes compared directly. The send side: each of several QPs' rate limiters runs entirely independent state -- current rate, target rate, alpha, timer, counter -- with no shared gate between them at all, unlike TX Scheduling or CQE Generation. The receive side: many simultaneously congested flows all needing a CNP at once, funneling through one bounded CNP-generation capability -- a real, measured ceiling (10-20 flows on one documented generation of hardware, 200+ on a later one), not a hypothetical arbitration question.](/images/congestion/08-multi-flow-cnp-arbitration-block.svg)

## Deep walkthrough

**Why the send side doesn't need this chapter at all.** [Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling), [Event and Multi-CQ Arbitration](/cq/event-and-multi-cq-arbitration), and [Multi-QP Error Arbitration](/qp/multi-qp-error-arbitration) all share the same shape: many QPs need the same single, serialized resource at overlapping moments, and something has to pick an order. Rate limiting has no equivalent chokepoint. Every QP's own rate-limiter context -- current rate, target rate, reaction factor, timer, byte counter -- is independent of every other QP's, checked and updated on its own schedule, gating only that QP's own dispatch. A hundred QPs all needing a rate-check at the same instant aren't contending for anything; they're each consulting their own, separate state.

> [!NOTE] Concept
> Not every resource this book covers has a multi-instance arbitration question. This one doesn't, on the send side, for a structural reason: arbitration exists where many instances share one gate. Independent per-QP state, with no shared gate at all, has nothing to arbitrate.

**Where the real contention actually is.** [CNP Generation](/congestion/cnp-generation) already named the fact this chapter builds on: generating a CNP is not free, and a real NIC's own capability to do it is bounded. One documented generation of shipped hardware supports the CNP generation rate this Part's own sourced algorithm requires for roughly 10 to 20 simultaneously congested flows; a later generation from the same vendor supports it for over 200. Both are real, measured figures from a named implementation, not this book's own estimate -- the same standard of evidence [QP Caching](/qp/qp-caching) and [Key Caching](/mr/key-caching) already used for their own working-set claims.

> [!TIP] Why
> This capacity being finite isn't a design flaw -- generating a CNP costs real work: parsing the marked packet, checking the per-flow notification window [CNP Generation](/congestion/cnp-generation) already covers, building and queuing a new outgoing packet. A NIC handling traffic at line rate has a bounded budget for that work, the same way [QP Caching](/qp/qp-caching)'s on-chip working set is bounded by scarce SRAM rather than an arbitrary design choice. What differs here is that this bound is measured and cited directly, not reasoned toward the way this book had to for QP context's own cache sizing.

**What happens when demand exceeds it -- genuinely open, and worth naming as such.** No source this book found describes what happens when more flows are simultaneously congested than one NIC's CNP-generation capability can service within the notification window [CNP Generation](/congestion/cnp-generation) already covers. Candidate outcomes are easy to imagine -- some flows' marks go unanswered until capacity frees up, generation falls back to a coarser or delayed schedule under load, or some flows are serviced preferentially over others -- but this is exactly the same honest position [Event and Multi-CQ Arbitration](/cq/event-and-multi-cq-arbitration) already took for its own two arbitration questions: this book doesn't have a citation pinning down which of these a real RNIC actually does.

> [!IMPORTANT] Food for Thought
> A large-scale incast -- the same scenario [CQ Overflow](/cq/cq-overflow) and [Multi-QP Error Arbitration](/qp/multi-qp-error-arbitration) both treat as the moment their own worst cases actually happen -- is also exactly the scenario most likely to congest many flows at once. If CNP-generation capacity is exhausted precisely when an incast is underway, what would that do to how quickly the flows involved actually converge on a fair rate, compared to a smaller-scale congestion event this bound never gets tested against?

**Where this connects.** This chapter closes the question [CNP Generation](/congestion/cnp-generation) opened but didn't answer: what happens at the boundary of that structure's own capacity. What happens to a QP's rate-limiter entry when it's evicted, rather than merely under-serviced, is [Eviction Policy](/congestion/eviction-policy), next.

## References

- Zhu, Eran, Firestone, Guo, Lipshteyn, Liron, Padhye, Raindel, Haj Yahia, Zhang, "Congestion Control for Large-Scale RDMA Deployments," SIGCOMM 2015 -- the ConnectX-3 Pro (10-20 flows) and ConnectX-4 (200+ flows) CNP-generation capacity figures, quoted directly.
- No IETF or IBTA standard, and no source this book found, describes what happens when demand for CNP generation exceeds a real implementation's own capacity -- this chapter states that gap directly rather than filling it with an invented answer.
