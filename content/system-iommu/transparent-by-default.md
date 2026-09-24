---
title: "Transparent by Default"
weight: 1
date: 2026-09-16
publishDate: 2026-09-16
draft: false
description: "Every 'physical address' this book has named so far is physical from this engine's own Key Cache's vantage point. Whether it's also the address that actually lands on a DRAM row depends on a layer this engine has no architectural awareness of at all -- and, by default, no way to find out."
next: /system-iommu/iotlb-miss-latency
---

[Address Translation Handoff](/dma/address-translation-handoff) named the output of a Key Cache lookup a physical address and moved on, the same term [Key Caching](/mr/key-caching) and [MR Capacity and Timing](/mr/mr-capacity-and-timing) both already used. That term is correct as far as those chapters' own scope reaches. This chapter is about what's true one layer further out.

## Overview

The address [Address Translation Handoff](/dma/address-translation-handoff) resolves is this engine's own endpoint, not necessarily the last translation a byte passes through before it reaches a DRAM row. In any virtualized, SR-IOV, or PCIe-passthrough deployment, a system-level IOMMU or SMMU sits on the path between this engine and real memory, and retranslates that address again -- transparently, by default, with this engine's own hardware carrying no architectural awareness that the retranslation is even happening.

> [!NOTE] Concept
> Call the address this engine's own Key Cache resolves to an IOVA -- an I/O virtual address, standard terminology for exactly this layer -- rather than assuming "physical" always means "final." This engine's own architecture never had to distinguish the two before this chapter; every prior chapter's use of "physical address" stays accurate for what it was describing, just scoped one layer short of the whole path.

## Locator

![The master RNIC block diagram with DMA Engine and System IOMMU / SMMU both highlighted -- the new external block added to this diagram, and the one place this engine's own output crosses a boundary this book hasn't covered before.](/images/system-iommu/01-transparent-by-default-locator.svg)

System IOMMU / SMMU is drawn the same way Host Memory Descriptor Rings and Host: Event/Interrupt already are -- external, dashed border, something outside the RNIC this engine's own pipeline directly touches. Unlike those two, it isn't guaranteed to exist in every deployment at all.

## Block diagram

![DMA Engine's own resolved IOVA crossing a dashed boundary into System IOMMU / SMMU, which retranslates it against its own page tables into a host physical address before it ever reaches DRAM -- everything past the boundary drawn dimmed and out of this engine's own visibility, with a second, dashed path showing the same IOVA passing straight through untouched when no system-level translation layer is present at all.](/images/system-iommu/01-transparent-by-default-block.svg)

## Deep walkthrough

**Why this layer exists at all, from this engine's own motivating angle.** A generic "why does an SoC need an IOMMU" case is a different site's territory. This book's own version of that question is sharper, because RDMA's whole model makes the answer more load-bearing than it is for most other DMA-capable devices: a WRITE lets a remote peer place data at an address this side's software never chose at the moment the packet arrived, and [Target Selection](/dma/target-selection) already established that mechanism in full. An RNIC whose virtual function is assigned directly to a guest -- SR-IOV, PCIe passthrough -- with no hypervisor mediating each individual operation, means a system-level IOMMU or SMMU's enforcement is the only thing standing between that guest's own RDMA traffic and a different tenant's physical memory. This engine's own Key Cache checks permission and bounds against what one QP's software registered; it has no way to check, and no reason to check, whether that registration itself was scoped correctly by a hypervisor that trusts the guest less than the guest trusts itself.

> [!TIP] Why
> Every other DMA-capable device on a bus benefits from system-level isolation. An RNIC benefits from it more, specifically because [The Operation Catalog](/overview/operation-catalog)'s own opcode set includes WRITE -- an operation type most bus masters never have an equivalent of, where the device itself, not local software, decides the destination address for an arriving payload. That's exactly the capability a compromised or misconfigured guest would want to abuse, and exactly the capability this layer exists to contain even when this engine's own checks pass cleanly.

**Transparent by default -- this engine's own architecture doesn't change.** Without any further mechanism, this engine issues the IOVA [Address Translation Handoff](/dma/address-translation-handoff) resolved exactly the same way whether a system-level IOMMU or SMMU is present or not. The retranslation, when it happens, happens at the boundary between the PCIe fabric and the memory controller -- outside this engine, outside the RNIC entirely -- and nothing in this engine's own execution model branches on whether that boundary exists. [In-Flight Transfer State](/dma/in-flight-transfer-state)'s current-address field holds this engine's own IOVA throughout a transfer's execution; it was never holding a guaranteed-final physical address, even before this chapter named the distinction.

> [!CAUTION] Gotcha
> Don't read this chapter as correcting an error in [Address Translation Handoff](/dma/address-translation-handoff), [Key Caching](/mr/key-caching), or [MR Capacity and Timing](/mr/mr-capacity-and-timing). None of them claimed anything about a system-level translation layer, and "physical address" is the right term for what each of them was describing -- this engine's own resolved endpoint. This chapter adds a layer those chapters never claimed to reach, rather than fixing a mistake in them.

**What "transparent" doesn't mean: free.** Transparency here describes this engine's own architecture, not the cost of what's happening underneath it. A retranslation that this engine has no way to observe can still take real time and can still fail -- [IOTLB Misses and the Latency This Engine Can't See](/system-iommu/iotlb-miss-latency), immediately next, is what that actually costs. [ATS: Caching a Translation Locally](/system-iommu/ats-local-caching) is the one real mechanism that trades this section's default transparency for visibility, on request.

**A resource this book's own multi-instance question doesn't reach.** [Multi-QP Burst Arbitration](/dma/multi-qp-burst-arbitration) already covers contention for this engine's own concurrent-transfer capacity. Whatever queueing or contention exists inside a system-level IOMMU or SMMU's own translation hardware -- shared, in a real deployment, across every device on the bus, not just this one RNIC -- is a different resource this engine has no visibility into and this book doesn't extend its own arbitration treatment to cover.

## Interface reference

| Question | Answer |
|---|---|
| **Is a system-level IOMMU/SMMU guaranteed to be present** | no -- deployment-dependent (virtualized, SR-IOV, or passthrough systems are the common case where one is) |
| **What does this engine's own resolved address represent precisely** | an IOVA -- this engine's own translation's endpoint, not necessarily the final DRAM address |
| **Does this engine's own architecture change based on whether one is present** | no, by default -- fully transparent, no awareness at either end of a transfer |
| **What does transparency cost** | nothing architecturally, but not nothing in latency -- see [IOTLB Misses and the Latency This Engine Can't See](/system-iommu/iotlb-miss-latency) |
| **Is there a way to opt out of the transparency and see the translation** | yes -- [ATS](/system-iommu/ats-local-caching), on request |

## References

- PCI-SIG, *PCI Express Base Specification*, Address Translation Services (ATS) extension (introduced PCIe 4.0) -- governs the one opt-in mechanism, covered in [ATS: Caching a Translation Locally](/system-iommu/ats-local-caching), that changes this chapter's default transparency.
- No IBTA or RoCEv2 standard governs system-level address translation at all -- it is host/root-complex architecture, entirely outside RDMA's own wire-level or QP-level scope. This chapter's own claims are architectural reasoning about the RNIC-facing half of a boundary those standards never address, not a citation to either standard.
