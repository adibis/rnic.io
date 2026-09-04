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

<!-- IMAGE: Master RNIC block diagram (see docs/MASTER_DIAGRAM_INVENTORY.md), full canvas, all blocks at opacity 0.35 except QP Table and QP Cache (full opacity, indigo highlight override per docs/SVG_STYLE.md, not their normal amber role color) plus the CQE Gen / CQ Storage block pair, also drawn at full opacity with the same indigo override, with a full-opacity connector arrow running from QP Table/Cache to CQE Gen -- this chapter's subject is specifically the shared path between many faulting QPs and the one reporting structure they all converge on. -->

This chapter is honest about its own limits before it starts: no vendor publishes how its RNIC actually schedules or prioritizes fault reporting when many QPs go down at once. [Single-QP Error Handling](../single-qp-error-handling) is grounded in documented verbs-layer behavior throughout. This chapter is reasoned architectural analysis of a real problem that behavior creates at scale, not a report of one real chip's internal arbitration logic — flagged plainly here, and again where it matters below.

## Overview

The previous chapter's isolation guarantee holds for exactly one fault at a time. A link flap, a shared upstream resource failing, or any single event that takes out a batch of QPs simultaneously turns "one QP flushes its queues and reports through the CQ" into "many QPs all need to flush and report through structures that are, at some point, shared and serialized" — and something has to decide the order that happens in.

## Block diagram

<!-- IMAGE: A many-to-one funnel diagram: eight or so small "QP" boxes on the left, each with a small red fault-role-color glyph attached (per docs/SVG_STYLE.md), each with an arrow converging into a single narrow "reporting path" channel in the center (drawn as a queue/FIFO shape), which fans back out on the right into a single "CQE Gen" box. Annotate the narrow channel with a bracket labeled "serialization point: one path, many simultaneous demands." -->

## Deep walkthrough

**Why this is a real arbitration problem, not just single-QP error handling repeated.** Nothing about one QP's fault path requires touching a resource any other QP is using at the same moment — its send and receive queues, its ERROR transition, its flush sequence are all private to it. The reporting side is different: the async event mechanism and the CQ/completion-event path a QP's flush eventually surfaces through are shared, serialized structures, built to handle events arriving one at a time from the normal course of traffic. A shared failure that faults many QPs simultaneously puts many QPs' worth of flush completions and fault events onto that same narrow, serialized path at once — and whatever hardware decides to service or report first, some QP's fault notification is going to be delayed behind another's, purely as a consequence of there being one channel and many simultaneous demands on it.

**What a sane servicing order might look like — reasoned, not documented.** A strict FIFO-by-detection-order policy is simple to build and impossible to starve in the aggregate (everything is serviced eventually, in the order it arrived), but it has no notion that some QP's fault might matter more urgently to software than another's — a control-plane QP and a bulk-data QP faulting in the same batch are reported in whatever order hardware happened to detect them, not in an order reflecting which one an application would rather know about first. A priority-weighted policy fixes that by letting some QPs (or some traffic classes) jump the queue, at the real cost of a fairness problem: a steady stream of high-priority faults can, in principle, indefinitely delay reporting a lower-priority QP's fault, a genuine starvation risk any priority scheme has to reckon with, not merely a theoretical one. Neither is a free lunch, and this book isn't picking a winner — a real design's choice here depends on workload assumptions (how often do batched faults actually happen, and does the deployment have traffic classes worth distinguishing at all) this chapter can't make on the reader's behalf.

**This is the same shared-reporting-path question Part 7 answers for the general case.** Everything above is specific to QP faults funneling through the reporting path, but the underlying shape — many simultaneous producers, one serialized consumer/reporting structure, some policy has to pick an order — is exactly what Part 7 (Completion Queue / Event Management) covers for event arbitration generally, not just for QP faults. That Part owns the general mechanism; this chapter's job was establishing that QP faults are one real, concrete source of demand on it, and why that demand doesn't arrive one at a time the way single-QP error handling implicitly assumes.

> **Design rationale.** The honest design tension here is that the QP-fault path would like an arbitration policy tuned to *why* QPs fault in batches (a link event probably wants every affected QP reported fast and roughly together, since they share one root cause worth diagnosing once) while a truly general event-reporting path has to stay agnostic to the source of whatever it's arbitrating between. A real implementation either builds one general-purpose arbitration mechanism and accepts that it isn't specifically tuned to the batched-fault case, or builds fault reporting a dedicated fast path separate from routine completion traffic — trading extra hardware complexity for a guarantee that a fault storm doesn't wait behind a queue of ordinary, successful completions. Which side of that trade a given design takes isn't something public sourcing settles, and isn't something this book asserts as fact.

## Interface and timing reference table

| Question a real design has to answer | This chapter's position |
|---|---|
| Servicing/reporting order under simultaneous faults | Not publicly documented for any real RNIC — reasoned analysis only |
| FIFO-by-detection-order | Simple, starvation-free in aggregate, no urgency awareness |
| Priority-weighted | Urgency-aware, real starvation risk for low-priority QPs under sustained high-priority fault load |
| Shared vs. dedicated fault-reporting path | Open design question this book does not resolve; see Part 7 for the general event-arbitration mechanism |

[Eviction Policy](../eviction-policy) leaves the fault path behind and returns to the QP Cache from [QP Caching](../qp-caching)'s numbers: what decides which cached QP gets evicted when the working set runs out of room.
