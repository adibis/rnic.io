---
title: "Key Caching"
weight: 3
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "The RNIC's on-chip cache doesn't have an MTT-only section -- it's the exact same SRAM Part 1 already showed you caching QP context. A memory-region-heavy workload and a QP-heavy workload are fighting over the same shelf space, and this chapter has the measured numbers for what losing that fight costs."
prev: /mr/the-key-table
next: /mr/registering-a-memory-region
---

## Locator

![The master RNIC block diagram with only the Key Cache highlighted, positioned in front of the MR/Key Table, every other block and connector dimmed.](/images/mr/03-key-caching-locator.svg)

The Key Cache is the on-chip working set of MPT/MTT entries sitting in front of the MR/Key Table's DRAM/HBM-backed storage -- this chapter, in the master diagram above. It is drawn as its own box for the same reason [QP Caching](/qp/qp-caching) drew the QP Cache as its own box: each is the working set for a different structure's lookups. But the two boxes are not two pieces of silicon. They are the same SRAM.

## Overview

Every DMA the adapter issues against a memory region -- an incoming RDMA Write resolving an `rkey`, an outgoing operation resolving an `lkey` -- needs a translated physical address before the transfer can happen. Resolving that translation from the MR/Key Table's backing store on every DMA would put a DRAM or PCIe round trip on the critical path of every transfer, the same problem [QP Caching](/qp/qp-caching) already described for QP context. The Key Cache exists for the same reason the QP Cache does: to make a translation that's actively in use resolve on-chip, and to make one that isn't cost a measurable, bounded penalty. What this chapter adds that [Part 1](/qp/) (Queue Pair Management)'s version didn't need to say explicitly: the SRAM doing that job for MTT/MPT is the identical SRAM doing it for QP context, and the two working sets are drawing from one shared, finite pool, not two separately sized ones.

## Block diagram

<!-- IMAGE: Zoomed diagram of the shared on-chip cache's internals. A single SRAM boundary box containing four labeled sub-clusters of small entry boxes side by side, all drawn in the same neutral slate fill to emphasize they occupy one undivided pool: "QPC entries" (indigo-tinted label, cross-referencing qp-caching.md's block diagram), "MTT/MPT entries" (this chapter's subject, drawn with slightly heavier stroke to indicate emphasis), "CQC entries," and "EQC entries." No dividing wall is drawn between the four clusters -- a thin dashed (not solid) separator only, captioned "no fixed partition; occupancy determined by access pattern." One lookup path arrives from the left: a "DMA Engine -- Address Translation stage" box, labeled "resolving lkey/rkey to physical address," pointing into the MTT/MPT cluster specifically. Two outcomes branch right: a solid indigo "hit" arrow returning to DMA Engine; a red "miss" arrow (per SVG_STYLE.md's fault/error role color) dropping down to an "MR/Key Table (DRAM/HBM-backed)" box, with a dashed return arrow labeled "PCIe round trip, populate cache line" back up into the MTT/MPT cluster. -->

## Deep walkthrough

**What.** The uRDMA paper's cache model states this plainly, describing the RNIC processing unit's relationship to its own on-chip cache: "the RNIC PU directly interacts with the cache to query QP context metadata and memory region information from QP Context table and Memory Translation/protection Table (MTT) respectively... cache also needs to maintain other essential metadata, including Completion Queue Context (CQC) and Event Queue Context (EQC)." Four resource types, one cache. This directly corroborates [QP Caching](/qp/qp-caching)'s ConnectX-4/5 finding of a 2MB on-chip SRAM holding QP context -- it's the same 2MB, and QPC was never its only tenant. No source found gives MTT/MPT a separate capacity number, and none should be assumed: the 2MB figure is the shared-cache total, not an MTT-only partition carved out of it.

**Where checked.** The DMA Engine's Address Translation stage is where a physical address actually gets resolved from a key -- the same stage the master diagram inventory notes as "where [Part 2](/mr/) (Memory Region / Key Table)'s Key Cache actually gets consulted," reached whenever an RDMA operation's `lkey` or `rkey` needs to become a real address before a transfer can proceed.

**When it fails.** Two independently sourced numbers make the miss cost concrete rather than qualitative. The uRDMA paper directly measured it: registering a single 4KB memory region gives an average RTT of about 2.6 μs; scaling up to 2^20 registered regions (4GB total, enough to overflow the on-chip working set) raises average RTT to about 5.1 μs -- a roughly 2x increase, attributed to translations missing the cache and requiring a PCIe round trip to host memory. And Kong et al. (NSDI 2023), measuring a real NVIDIA ConnectX-5 RNIC directly, found that six threads driving 96 QPs against more than 18,000 registered memory regions degrade bandwidth by 40.1% through MTT cache misses alone -- a data-plane access pattern spreading wide enough across a large MR population to blow the cache's working set on real, shipped hardware.

**What happens next.** Because this cache is shared, not partitioned, pressure doesn't stay contained to whichever resource type caused it. A QP-heavy workload that pushes [QP Caching](/qp/qp-caching)'s working set past its own comfortable size isn't only generating QPC misses -- it's evicting MTT/MPT entries out of the exact same SRAM those QP contexts live in, and the reverse holds equally: a large registered-MR population competing for space can push active QP contexts out, degrading packet processing that has nothing to do with the memory region driving the eviction. What decides which entry gets evicted, from which resource type, under this shared pressure is chapters 2.7 and 2.8's subject; this chapter's job is establishing that the pressure is real, measured, and cross-resource.

> **Design rationale.** An RNIC could give QPC, MTT/MPT, CQC, and EQC each a fixed, dedicated slice of on-chip SRAM -- simpler to reason about, and no resource type can ever be starved by another's workload. Nobody ships that either, for the same reason no real cache anywhere partitions itself by data type when it doesn't have to: a fixed split wastes SRAM on whichever resource type a given deployment happens to use lightly, while starving the one it uses heavily, and the right ratio is workload-dependent in a way no single silicon design can fix at tape-out. Sharing the pool lets the cache's occupancy follow the actual access pattern instead of a static guess. The cost, made concrete by this chapter's numbers, is that a workload heavy in one resource type can degrade a completely unrelated one sharing the same SRAM -- a QP-count spike hurting MR translation performance, or vice versa, isn't a bug in either subsystem; it's the direct, unavoidable consequence of the sharing decision.

## Interface / timing reference

| Property | Value | Source / notes |
|---|---|---|
| On-chip SRAM (QPC + CQC + EQC + MTT/MPT, shared) | 2 MB | ConnectX-4/5, same shared-cache figure [QP Caching](/qp/qp-caching) cites -- not an MTT-specific partition |
| MTT/MPT-specific capacity split | not documented | no source gives MTT/MPT its own allocation; do not assume one |
| Average RTT, single 4KB MR registered | ~2.6 μs | uRDMA paper (arXiv 2510.00735), directly measured |
| Average RTT, 2^20 MRs registered (4GB, working set overflowed) | ~5.1 μs (~2x increase) | same source, same measurement |
| Bandwidth degradation, 18K+ MRs / 96 QPs / 6 threads | 40.1% | Kong et al., NSDI 2023, measured directly on ConnectX-5 |

The next chapter covers what happens before any of this: how a memory region gets into the table -- and, for one entry at a time, into this cache -- in the first place.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
- No IETF or IBTA standard governs on-chip MTT/MPT caching and working-set sizing -- it is implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above, not a standards document.
