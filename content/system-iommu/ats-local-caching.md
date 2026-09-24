---
title: "ATS: Caching a Translation Locally"
weight: 3
date: 2026-09-16
publishDate: 2026-09-16
draft: false
description: "Transparent by Default's default costs a full round trip to the system-level translation layer on every access this engine's own cache doesn't already know about. PCIe's own Address Translation Services extension is the one real mechanism that lets a device cache that answer locally instead -- on request, not by default."
prev: /system-iommu/iotlb-miss-latency
---

## Overview

[Transparent by Default](/system-iommu/transparent-by-default) established that this engine has no architectural awareness of a system-level IOMMU or SMMU at all. Address Translation Services -- ATS, a PCIe Base Specification extension maintained by PCI-SIG, introduced at PCIe 4.0 -- is the one real, standard mechanism that changes that on request: a device requests a translation explicitly, caches the answer locally, and uses the cached result directly on every later access to the same address, without repeating [IOTLB Misses and the Latency This Engine Can't See](/system-iommu/iotlb-miss-latency)'s own round trip each time.

## Locator

![The master RNIC block diagram with DMA Engine and System IOMMU / SMMU both highlighted -- this chapter is the one opt-in mechanism that turns this engine from a passive, uninformed issuer of addresses into an active participant holding its own cached copy.](/images/system-iommu/03-ats-local-caching-locator.svg)

## Block diagram

![A device-side Address Translation Cache (ATC) sitting between DMA Engine and the system-level IOMMU/SMMU: a request for a translation, a Translated Address response cached locally, every later access to the same IOVA resolving from the ATC directly without crossing the boundary again, and a separate invalidation path -- the system-level layer sending an Invalidate Request downstream and the device confirming with an Invalidate Completion once its own cached entry is actually cleared.](/images/system-iommu/03-ats-local-caching-block.svg)

## Deep walkthrough

**What ATS actually adds: a request-and-cache round trip, paid once instead of every time.** A device supporting ATS asks the system-level translation layer for a specific IOVA's translation explicitly, receives a Translated Address in response, and stores it in its own Address Translation Cache (ATC, sometimes called a Dev-TLB). Every later access to that same IOVA resolves from the device's own ATC directly -- the device issues the already-translated address itself, skipping the transparent-retranslation path [Transparent by Default](/system-iommu/transparent-by-default) already described, and skipping [IOTLB Misses and the Latency This Engine Can't See](/system-iommu/iotlb-miss-latency)'s own round trip on every one of those later accesses.

> [!NOTE] Concept
> An ATC is this engine's own equivalent of [Key Caching](/mr/key-caching)'s working set, one layer further out and for a different translation entirely -- system-level IOVA-to-host-physical, not this engine's own key-to-IOVA. The two caches solve the identical shape of problem -- a translation that's actively in use should resolve locally instead of costing a round trip every time -- at two different layers this book has now named separately rather than conflating.

**A different question from the one [Mid-Transaction Burst Teardown](/dma/mid-transaction-burst-teardown) left open.** That chapter asked whether this engine holds a private copy of its *own* Key-Cache-resolved translation for a burst's duration, or re-touches the shared Key Cache mid-burst -- a question about this engine's own MR/Key Table layer, still unresolved. An ATC is a real, standard-defined mechanism at a different layer entirely: caching the *system-level* translation, not this engine's own. Don't read this chapter as answering that earlier open question -- it's the same shape of problem recurring one layer out, with its own, separately-specified answer this time, not a resolution of the first one.

> [!WARNING] Analogy
> This is the same "physical-versus-logical, same shape at a different layer" pattern [The Rate Limiter Table](/congestion/the-rate-limiter-table) already drew for a different pair of resources. Two caches, two layers, two independent questions about what each one privately holds and for how long -- resolving one doesn't resolve the other.

**Coherency isn't left to chance -- ATS defines its own invalidation protocol.** A cached translation can go stale the same way any cache entry can: the mapping it reflects changes underneath it. ATS's own answer is a real, defined request-completion exchange -- the system-level translation layer sends an Invalidate Request downstream to the device's ATC, and the device is required to respond with an Invalidate Completion only once it has actually cleared the entry, not merely acknowledged the request. This is a real, standard-defined answer to the same shape of question [Mid-Transaction Deregistration](/mr/mid-transaction-deregistration) left open for this engine's own Key Cache -- what happens when the thing a cached translation depends on changes while that translation is still in use -- just settled explicitly, one layer out, by a mechanism this engine's own layer doesn't have.

> [!CAUTION] Gotcha
> Don't assume this engine, or any specific shipped RNIC, actually implements ATS. This chapter describes a real, available PCIe mechanism any capable device can use -- it is not a claim that RDMA NICs commonly do. No source this book found confirms ATS adoption specifically in shipped RNIC silicon either way; treat this chapter as naming what the mechanism offers, not asserting that this engine has it.

**The other real benefit ATS enables, named without extending this book's own scope to it.** ATS pairs with a separate PCIe mechanism, the Page Request Interface (PRI), that lets a device tolerate a translation not being resident yet -- requesting it on demand rather than requiring everything pinned in advance. That's architecturally the same shape of relaxation [Registering a Memory Region](/mr/registering-a-memory-region)'s own On-Demand Paging already described for this engine's Key Table layer, one layer further out and governed by PCI-SIG rather than IBTA. This chapter names the parallel and stops there -- PRI's own mechanics belong to PCIe's own specification, not this book's.

## Interface reference

| Question | Answer |
|---|---|
| **What does ATS add over the transparent default** | a local cache (ATC) of the system-level translation, requested explicitly |
| **What resolves an ATC hit** | the device's own cached Translated Address -- no round trip to the system-level layer |
| **Is this the same open question as Mid-Transaction Burst Teardown's private-copy question** | no -- a different layer, a different (standard-defined) answer |
| **How does a stale entry get cleared** | Invalidate Request (system-level layer) / Invalidate Completion (device), a defined round trip |
| **Is ATS confirmed to be implemented in shipped RNICs** | not confirmed either way by any source this book found |

## References

- PCI-SIG, *PCI Express Base Specification*, Address Translation Services (ATS) extension, introduced at PCIe 4.0 -- governs the request/response translation-caching mechanism and the Invalidate Request/Completion coherency protocol this chapter describes.
- No IBTA or RoCEv2 standard governs ATS or any system-level caching mechanism -- entirely PCIe/host-architecture territory, named here only for what it means to this engine's own default transparency, not re-derived from PCI-SIG's own specification text.
