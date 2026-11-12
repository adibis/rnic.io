---
title: "System IOMMU / SMMU"
weight: 11
description: "Every address this book has called 'physical' so far is physical from this engine's own vantage point -- not necessarily the last translation a byte passes through before it reaches DRAM. This topic covers the RNIC-facing half of that boundary: whether a system-level IOMMU or SMMU is visible to this engine at all, what an IOTLB miss it has zero visibility into actually costs, and whether ATS lets it cache a translation locally instead of paying that cost twice."
icon: shield-check
cascade:
  type: docs
metaDescription: "How a system IOMMU or SMMU sits between an RDMA NIC and memory: transparent by default, IOTLB misses the NIC cannot see, and caching with PCIe ATS."
seoTitle: "RDMA NICs and the System IOMMU or SMMU"
homeSummary: "The second translation past PCIe, IOTLB misses the NIC can't see, and ATS."
---

[Address Translation Handoff](/dma/address-translation-handoff) resolved a key to a physical address and moved on. That address is physical from this engine's own Key Cache's vantage point -- not necessarily the address that actually lands on a DRAM row. In any virtualized, SR-IOV, or PCIe-passthrough deployment, a system-level IOMMU or SMMU sits between this engine's own output and real memory, retranslating an address this book has been calling final. This topic covers the RNIC-facing half of that boundary -- not the IOMMU/SMMU's own internal mechanism, which is a different site's territory entirely, but what having one in the datapath means for this engine specifically: whether it's transparent, what a miss costs, and whether a real implementation has any way to avoid paying that cost on every single transfer.

## Chapters

{{< section-cards >}}
