---
title: "IOTLB Misses and the Latency This Engine Can't See"
weight: 2
date: 2026-09-16
publishDate: 2026-09-16
draft: false
description: "Transparent doesn't mean free. A system-level IOMMU or SMMU's own cache miss adds real, measured latency to a transfer this engine has already committed a slot to -- and this engine has no event, no status bit, and no way at all to learn that's what just happened."
prev: /system-iommu/transparent-by-default
next: /system-iommu/ats-local-caching
---

## Overview

[Transparent by Default](/system-iommu/transparent-by-default) established that this engine issues the same IOVA whether or not a system-level IOMMU or SMMU is present, and that the retranslation, when it happens, happens entirely outside this engine's own architecture. This chapter is about what that transparency doesn't promise: that the retranslation is free. A miss in the system-level translation cache costs real, measured time, on a transfer this engine has already committed one of its own concurrent-transfer slots to -- and this engine has no way to learn a miss is what just happened.

## Locator

![The master RNIC block diagram with DMA Engine and System IOMMU / SMMU both highlighted -- this chapter is about a cost this engine pays without a corresponding event or status this engine can read.](/images/system-iommu/02-iotlb-miss-latency-locator.svg)

## Block diagram

![A transfer's IOVA hitting the system-level IOMMU/SMMU's own translation cache -- a hit returning immediately, a miss dropping to a page-table walk on that layer's own side, adding measured latency before the same transfer can proceed, with a dashed boundary marking that neither outcome produces any signal this engine's own hardware can observe.](/images/system-iommu/02-iotlb-miss-latency-block.svg)

## Deep walkthrough

**A real, measured cost, not a theoretical one.** Research measuring IOMMU translation-cache behavior directly gives this chapter a concrete anchor: Tiemann et al. (2022) measured roughly 330 nanoseconds of added latency per IOTLB miss -- about 66 clock cycles on the 200MHz platform it measured. A separate, older line of IOTLB-bottleneck research (Amit, Ben-Yehuda, and Yassour, WIOSCA 2010) measured double-digit-percent execution-time penalties from IOTLB misses under I/O-intensive workloads -- their own reported figures range roughly from single digits up to around 50% depending on transfer size and access pattern, not a single clean number. This book states that range approximately rather than quoting one of their specific figures as if it were this chapter's own precise finding, since which exact number applies depends heavily on workload shape and neither paper's platform is this chapter's own. Neither number is this engine's own number -- both are properties of the system-level translation cache and whatever page-table structure backs it, entirely outside this book's own scope to specify -- but the order of magnitude is real and worth carrying forward: this isn't a rounding error, it's hundreds of nanoseconds to potentially double-digit-percent throughput, on a resource this engine has already spent one of its own limited concurrent-transfer slots holding.

> [!NOTE] Concept
> An IOTLB is the system-level IOMMU or SMMU's own translation cache -- the direct architectural analog of [Key Caching](/mr/key-caching)'s own working set, one layer further out. A hit resolves immediately; a miss falls back to a page-table walk on that layer's own side, the same shape of penalty [Key Caching](/mr/key-caching) already established for a Key Cache miss, just paid by a different structure this engine has no access to.

**Compounding, not replacing, this engine's own arbitration question.** [Multi-QP Burst Arbitration](/dma/multi-qp-burst-arbitration) already named transfer-duration variance as a real head-of-line risk -- a long, multi-segment gather holding a concurrent-transfer slot for longer than a short one ever would. An IOTLB miss hidden inside that same slot's occupancy makes the problem worse in a way this engine's own arbitration policy has no way to anticipate or account for: two transfers identical in segment count and length can occupy a slot for measurably different durations, for a reason entirely outside this engine's own visibility.

> [!CAUTION] Gotcha
> Don't reach for [Single-Burst Fault Handling](/dma/single-burst-fault-handling)'s fault classes to explain a transfer that's simply running slow. An IOTLB miss is not a fault by that chapter's own definition -- the transfer still completes successfully, on the same data, just later. It produces no error status, no QP-to-ERROR transition, nothing at all that this engine's own completion-reporting path would ever surface. Whether an IOVA the system-level layer genuinely cannot map at all -- as opposed to a routine miss followed by a successful walk -- produces a real, distinct failure this engine would observe is a question this book found no source addressing either way.

**Why this engine can't simply be told.** Nothing about the PCIe transaction layer or the interfaces this book has already covered gives a routine memory read or write a side-channel for "this took longer than usual, and here's why." A transfer's completion, when it finally lands, looks identical whether it resolved instantly or paid a full page-table walk somewhere upstream of this engine's own boundary. This isn't a gap this book can name a specific missing mechanism for -- it's a direct, structural consequence of the transparency [Transparent by Default](/system-iommu/transparent-by-default) already established: an architecture that doesn't expose the boundary at all can't selectively expose what happened at it, either.

**Where a real implementation would feel this first.** [Ordering and Outstanding-DMA Tracking](/dma/ordering-and-outstanding-dma-tracking) already named this engine's own concurrent-transfer capacity as a fixed, implementation-specific resource. A workload whose IOVAs miss the system-level translation cache often enough -- a large, sparse memory footprint relative to that cache's own size, the same shape of pressure [Key Caching](/mr/key-caching) already described for its own resource -- would show up as this engine's own slots staying occupied longer on average, with no corresponding change in this engine's own reported activity to explain why.

## Interface reference

| Question | Answer |
|---|---|
| **Does a miss produce an error or status this engine can read** | no -- fully invisible, per Transparent by Default |
| **Is the added latency real** | yes -- measured at roughly 330ns / 66 cycles per miss in one study; roughly single digits to ~50% execution-time penalty under I/O-intensive workloads in another, depending on transfer size and pattern |
| **Does it count as a fault** | no -- the transfer still completes; distinct from every class in Single-Burst Fault Handling |
| **What does it compound with** | Multi-QP Burst Arbitration's own head-of-line risk -- a slot held longer, for a reason this engine can't see |
| **Is an unmappable IOVA (not just a slow one) a different case** | plausibly, but no source found describes what this engine would observe if so |

## References

- Tiemann et al. (2022), on IOTLB timing behavior -- the ~330ns / 66-cycle per-miss figure on a 200MHz FPGA platform, a real, measured anchor for this chapter's central claim, sourced via secondary research rather than a direct read of the paper's own text.
- Amit, Ben-Yehuda, and Yassour, "IOMMU: Strategies for Mitigating the IOTLB Bottleneck," WIOSCA 2010 -- the source for the double-digit-percent execution-time-penalty figures cited above. Their own reported numbers vary by transfer size and access pattern rather than converging on one figure; this chapter states that range approximately rather than attributing one specific number to it as a precise finding.
- No IBTA, RoCEv2, or PCIe standard obligates a device to be informed of a translation-layer cache miss -- this chapter's central claim (this engine has no way to observe one) is architectural reasoning from the transparency Transparent by Default already established, not a citation to an explicit "no notification" rule.
