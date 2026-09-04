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

Every packet that touches a QP — building one on transmit, parsing one on receive — needs that QP's live context: its state, its PSN counters, its negotiated path. Fetching that context from the QP Table on every single packet would mean a DRAM or PCIe round trip on the critical path of every packet the adapter handles, which is not compatible with line-rate operation. The QP Cache exists to make the common case — a QP that's actively carrying traffic — resolve on-chip, and to make the uncommon case — a QP that isn't — cost a measurable, bounded penalty instead of an unbounded one.

## Block diagram

<!-- IMAGE: Zoomed diagram of QP Cache internals only. A cluster of small boxes labeled "QPC" (~375 bytes each, per the cited figure below), captioned collectively "on-chip SRAM working set." Two lookup paths arrive from the left into a single lookup point on the cache: one from a "TX Scheduler" box labeled "QP being scheduled," one from an "RX Parser" box labeled "destination QP from BTH." Two outcomes branch right from the lookup point: a solid indigo "hit" arrow returning directly to whichever pipeline stage asked; a red "miss" arrow (per SVG_STYLE.md's fault/error role color) dropping down to a "QP Table (DRAM/HBM-backed)" box, with a dashed return arrow labeled "PCIe fetch, populate cache line" back up into the cache. Keep the cache's internal box generic ("working set of cached QPCs") rather than depicting a specific tag/index/associativity structure — that organization is not publicly documented for any real RNIC and should not be drawn as if it were. -->

## Deep walkthrough

**What.** The cache holds a working set of QP Contexts (QPCs) — the same per-QP fields chapter 1.1 covers field by field, populated from the QP Table entry the last time that QP was touched. A cited 2026 study of QP-scalable RDMA loss recovery in hyperscale AI training clusters measured a Reliable Connection QPC at roughly 375 bytes on Mellanox ConnectX-4/5 hardware, against 2MB of on-chip SRAM available to hold address mappings and QP metadata together — which puts the fully-cacheable working set at around 5,000 QPs before the SRAM is full and something has to be evicted to make room for anything new.

**Where checked.** Every pipeline stage that needs a QP's context checks the cache first: the TX Scheduler, deciding which QP to service next, and the RX Parser, resolving the destination QP number it just read out of an incoming BTH (0.2). Neither stage has any path to the wire or to host memory that skips this lookup.

**When it fails.** A miss means the requested QPC isn't currently resident in the on-chip working set, and the same cited study found this is not a rare corner case at scale: ConnectX-6 hardware, documented to address up to 131,072 QPs, was measured losing throughput past roughly 256 concurrently active QPs specifically because of cache misses forcing PCIe round trips back to host memory to retrieve QP metadata. That gap — 131,072 addressable against a working set that collapses two orders of magnitude lower — is not a defect in one generation of silicon; it's the direct, structural consequence of on-chip SRAM being small, expensive, and shared with other jobs (address translation lives in the same 2MB in that same study), against a QP count that scales with however many connections an application layer decides to open.

**What happens next.** A miss doesn't drop or corrupt the packet waiting on it — it stalls that packet's processing behind a fetch, at PCIe latency, which is measured in microseconds against an on-chip hit's nanoseconds-scale cost. Enough concurrently active QPs beyond the cacheable working set, and enough of the traffic touching QPs outside that set, and the adapter spends more time fetching context than building packets — which is exactly the throughput collapse the cited numbers above describe. What decides which QPC gets evicted to make room for an incoming one, and whether that decision is fixed in hardware or programmable, is chapters 1.8 and 1.9's entire subject; this chapter's job is establishing that the decision has to be made at all, and how expensive getting it wrong actually is.

> **Design rationale.** An RNIC could, in principle, skip caching entirely and hit the QP Table on every packet — simpler hardware, no working-set question to answer, no eviction policy to design. Nobody ships that design, because the cost is a DRAM or PCIe transaction sitting on the critical path of every single packet, at every port, all the time, rather than only on the packets touching a QP outside the current working set. The tradeoff a real implementation actually makes isn't "cache or don't" — it's how much scarce, expensive on-chip SRAM to spend on QP context specifically, against every other structure (address translation, buffering) competing for the same die area, knowing that whatever number gets chosen, some real deployment will eventually run more concurrently-active QPs than it can hold.

## Interface / timing reference

| Property | Value | Source / notes |
|---|---|---|
| On-chip SRAM (QP metadata + address mappings, shared) | 2 MB | ConnectX-4/5, cited 2026 study — one generation's figure, not a universal constant |
| RC QPC size | ~375 bytes | same source |
| Fully-cacheable working set | ~5,000 QPs | derived from the two figures above, same source |
| Addressable QP count | up to 131,072 | ConnectX-6 documented capability |
| Empirical throughput-collapse point | ~256 concurrently active QPs | ConnectX-6, same source — cache misses forcing PCIe round trips |
| Hit cost | on-chip, nanoseconds-scale | qualitative — no cited cycle count |
| Miss cost | PCIe round trip to host memory, microseconds-scale | qualitative — no cited cycle count |

The next chapter covers what happens when a QP doesn't yet exist in either the cache or the table at all — the software-to-hardware handoff that creates one.
