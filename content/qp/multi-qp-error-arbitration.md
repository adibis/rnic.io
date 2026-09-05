---
title: "Multi-QP Error Arbitration"
weight: 7
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "One QP faulting is a contained event with a clean isolation boundary. A hundred QPs faulting in the same instant, from the same shared cause, all need to tell software about it through the same narrow channel -- and no public source says how a real RNIC decides who goes first."
prev: /qp/single-qp-error-handling
next: /qp/eviction-policy
---

## Locator

![The master RNIC block diagram with QP Table, QP Cache, CQE Gen, and CQ Storage all highlighted -- this chapter's subject is the shared reporting path many faulting QPs converge on.](/images/qp/07-multi-qp-error-arbitration-locator.svg)

This chapter is honest about its own limits before it starts: no vendor publishes how its RNIC actually schedules or prioritizes fault reporting when many QPs go down at once. [Single-QP Error Handling](../single-qp-error-handling) is grounded in documented verbs-layer behavior throughout. This chapter is reasoned architectural analysis of a real problem that behavior creates at scale, not a report of one real chip's internal arbitration logic — flagged plainly here, and again where it matters below.

## Overview

Multi-core RAS design faces the same shape of question when a resource shared across cores -- a cache bank, an interconnect ring -- faults and takes several of them down at once: containment per core is straightforward to reason about one at a time, and gets much harder the instant one cause faults many simultaneously. The previous chapter's isolation guarantee holds for exactly one fault at a time. This isn't only a hypothetical "link flap" scenario -- it's a documented, concrete mechanism: when a Completion Queue or Shared Receive Queue itself hits an error, *every* QP associated with it transitions to ERROR and fires `IBV_EVENT_QP_FATAL` simultaneously, not one at a time ([Single-QP Error Handling](../single-qp-error-handling) names this trigger directly). A CQ or SRQ shared across dozens or hundreds of QPs turns one upstream failure into exactly that many simultaneous fault events, all needing to flush and report through structures that are, at that instant, shared and serialized — and something has to decide the order that happens in.

## Block diagram

![A shared CQ or SRQ that has itself hit an error fires IBV_EVENT_QP_FATAL on eight associated QPs at the same instant, all funneling down through one serialized reporting path into CQE Gen.](/images/qp/07-multi-qp-error-arbitration-block.svg)

## Deep walkthrough

**Why this is a real arbitration problem, not just single-QP error handling repeated.** Nothing about one QP's fault path requires touching a resource any other QP is using at the same moment — its send and receive queues, its ERROR transition, its flush sequence are all private to it. The reporting side is different: the async event mechanism and the CQ/completion-event path a QP's flush eventually surfaces through are shared, serialized structures, built to handle events arriving one at a time from the normal course of traffic. A shared failure that faults many QPs simultaneously puts many QPs' worth of flush completions and fault events onto that same narrow, serialized path at once — and whatever hardware decides to service or report first, some QP's fault notification is going to be delayed behind another's, purely as a consequence of there being one channel and many simultaneous demands on it.

**What a sane servicing order might look like — reasoned, not documented.** A strict FIFO-by-detection-order policy is simple to build and impossible to starve in the aggregate (everything is serviced eventually, in the order it arrived), but it has no notion that some QP's fault might matter more urgently to software than another's — a control-plane QP and a bulk-data QP faulting in the same batch are reported in whatever order hardware happened to detect them, not in an order reflecting which one an application would rather know about first. A priority-weighted policy fixes that by letting some QPs (or some traffic classes) jump the queue, at the real cost of a fairness problem: a steady stream of high-priority faults can, in principle, indefinitely delay reporting a lower-priority QP's fault, a genuine starvation risk any priority scheme has to reckon with, not merely a theoretical one. Neither is a free lunch, and this book isn't picking a winner — a real design's choice here depends on workload assumptions (how often do batched faults actually happen, and does the deployment have traffic classes worth distinguishing at all) this chapter can't make on the reader's behalf.

**This is the same shared-reporting-path question Part 7 (Completion Queue / Event Management) answers for the general case.** Everything above is specific to QP faults funneling through the reporting path, but the underlying shape — many simultaneous producers, one serialized consumer/reporting structure, some policy has to pick an order — is exactly what Part 7 (Completion Queue / Event Management) covers for event arbitration generally, not just for QP faults. That Part owns the general mechanism; this chapter's job was establishing that QP faults are one real, concrete source of demand on it, and why that demand doesn't arrive one at a time the way single-QP error handling implicitly assumes.

> **Design rationale.** The honest design tension here is that the QP-fault path would like an arbitration policy tuned to *why* QPs fault in batches (a link event probably wants every affected QP reported fast and roughly together, since they share one root cause worth diagnosing once) while a truly general event-reporting path has to stay agnostic to the source of whatever it's arbitrating between. A real implementation either builds one general-purpose arbitration mechanism and accepts that it isn't specifically tuned to the batched-fault case, or builds fault reporting a dedicated fast path separate from routine completion traffic — trading extra hardware complexity for a guarantee that a fault storm doesn't wait behind a queue of ordinary, successful completions. Which side of that trade a given design takes isn't something public sourcing settles, and isn't something this book asserts as fact.

[Eviction Policy](../eviction-policy) leaves the fault path behind and returns to the QP Cache [QP Caching](../qp-caching) already established: what decides which cached QP gets evicted when the working set runs out of room.

## References

No IETF or IBTA standard governs multi-QP fault servicing/reporting order -- it is implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above (the `IBV_EVENT_QP_FATAL` cascading-fault mechanism is documented verbs-layer behavior; the servicing-order reasoning is not), not a standards document.
