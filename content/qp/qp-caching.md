---
title: "QP Caching"
weight: 3
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "A NIC will sell you far more queue pairs than it can actually keep fast. This chapter is about the gap between those two numbers, and the on-chip cache that decides which QPs live on the right side of it."
prev: /qp/the-qp-table
next: /qp/installing-a-qp
---

## Locator

![The master RNIC block diagram with only the QP Cache highlighted, positioned in front of the QP Table, every other block and connector dimmed.](/images/qp/03-qp-caching-locator.svg)

The QP Cache is the on-chip working set of QP contexts sitting in front of the QP Table's DRAM/HBM-backed storage — this chapter, in the master diagram above.

## Overview

A CPU never re-walks a page table on every memory access it can avoid -- a TLB caches the translations actually in use, because paying DRAM/page-walk latency on every single access would be unworkable at core clock speeds. A QP's live context faces the identical problem for a different structure: every packet that touches a QP — building one on transmit, parsing one on receive — needs that QP's live context: its state, its PSN counters, its negotiated path. Fetching that context from the QP Table on every single packet would mean a DRAM or PCIe round trip on the critical path of every packet the adapter handles, which is not compatible with line-rate operation. The QP Cache exists to make the common case — a QP that's actively carrying traffic — resolve on-chip, and to make the uncommon case — a QP that isn't — cost a measurable, bounded penalty instead of an unbounded one.

## Block diagram

![The QP Cache with hit and miss paths: TX Scheduler and RX Parser both look up the cache first, a hit returns directly, a miss falls through to the QP Table with a PCIe fetch to repopulate the cache line.](/images/qp/03-qp-caching-block.svg)

## Deep walkthrough

**What the cache holds.** A working set of QP Contexts (QPCs) — the same per-QP fields [QP Context](../qp-context) covers field by field, populated from the QP Table entry the last time that QP was touched. The on-chip SRAM holding this working set is scarce and expensive by construction, the same constraint that governs any on-chip cache -- this book doesn't have a citable source pinning down an exact QPC byte size or exact SRAM budget for any specific chip, and doesn't state one as fact. What is real and citable: "Communication-Semantic-Aware RDMA Loss Recovery for QP-scalable Hyperscale AI Training" (arXiv 2606.20582) frames the identical problem this chapter opens with -- a real RNIC can only keep a bounded working set of QP contexts resident on-chip at once, nowhere near the full range of QPs the adapter can address, which is exactly the gap this chapter's "hit" and "miss" cases exist to describe.

**Where this is checked.** Every pipeline stage that needs a QP's context checks the cache first: the TX Scheduler, which consults it to decide which QP to service next, and the RX Parser, which consults it to resolve the destination QP number it just read out of an incoming BTH (0.2). Neither stage has any path to the wire or to host memory that skips this lookup.

**When a lookup misses.** A miss means the requested QPC isn't currently resident in the on-chip working set, and the same paper treats this as a real, non-rare condition at scale, not a hypothetical corner case: a real, shipped RNIC's cache-bounded QP working set means traffic touching enough QPs beyond that bound pays a PCIe round trip back to host memory to retrieve QP metadata, on real tested hardware. That gap between what a real adapter can address and what its cache can actually hold resident is not a defect in one generation of silicon; it's the direct, structural consequence of on-chip SRAM being small and expensive, against a QP count that scales with however many connections an application layer decides to open. Whether that SRAM is dedicated to QP context alone or shared with other on-chip structures (address translation, per [Key Caching](/mr/key-caching), is one resource confirmed to share its own on-chip pool with other state) isn't something this specific paper documents for QP context -- this book doesn't extend that MR-side finding to the QP cache without its own source.

**What happens after a miss.** A miss doesn't drop or corrupt the packet waiting on it — it stalls that packet's processing behind a fetch, at PCIe latency, against an on-chip hit's much cheaper cost. Enough concurrently active QPs beyond the cacheable working set, and enough of the traffic touching QPs outside that set, and the adapter spends more time fetching context than building packets — a real, structural consequence of a bounded working set, not a hypothetical one. What decides which QPC gets evicted to make room for an incoming one, and whether that decision is fixed in hardware or programmable, is chapters 1.8 and 1.9's entire subject; this chapter's job is establishing that the decision has to be made at all, and that getting it wrong carries a real cost.

> **Design rationale.** An RNIC could, in principle, skip caching entirely and hit the QP Table on every packet — simpler hardware, no working-set question to answer, no eviction policy to design. The cost of that choice is a DRAM or PCIe transaction sitting on the critical path of every single packet, at every port, all the time, rather than only on the packets touching a QP outside the current working set -- which is exactly the cost caching exists to avoid. The tradeoff a real implementation actually makes isn't "cache or don't" — it's how much scarce, expensive on-chip SRAM to spend on QP context specifically, against every other structure (address translation, buffering) competing for the same die area, knowing that whatever amount gets chosen, some real deployment will eventually run more concurrently-active QPs than it can hold.

[Installing a QP](../installing-a-qp) covers what happens when a QP doesn't yet exist in either the cache or the table at all — the software-to-hardware handoff that creates one.

## References

- No IETF or IBTA standard governs on-chip QP context caching and working-set sizing -- it is implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above, not a standards document.
