---
title: "Overview"
weight: 1
description: "What RDMA demands of a NIC, RoCEv2's wire format at overview depth, and one transaction walked start to finish -- the map this book's block-by-block parts fill in afterward."
icon: book-open
cascade:
  type: docs
---

A conventional NIC moves frames between a wire and a ring buffer and stops there. An RDMA-capable NIC takes a virtual address and a remote key and moves bytes into another machine's memory directly -- no syscall, no remote CPU, no unauthorized byte moved. This part names that shift, lays out the RoCEv2 wire format that carries it, and walks one operation through the whole machine. Every stage that operation touches gets its own part later in the book.

## The whole chip, in one picture

Below is the same diagram every chapter in this book opens with -- normally with just one box lit up to show where that chapter fits. Here it's shown whole, before anything gets zoomed in on.

The dashed boxes on the left and right sit in the host machine, not on the RNIC chip itself -- that's where an operation starts and where it eventually gets reported back. The amber boxes store or look something up (a connection's state, an address translation, an unacknowledged packet waiting to be resent). The blue boxes are where a packet actually gets built, parsed, or moved. Everything else is just the wiring between them.

Boxes that already have a chapter written are clickable -- click one to go straight there. A few boxes on the right (Rate Limiter, DMA Engine, CQE Gen, CQ Storage) don't have their chapters yet, so they're not links.

---

<div class="master-diagram-wrap master-diagram-light">
{{< inline-svg "static/images/overview/00-master-diagram-linked.svg" >}}
</div>
<div class="master-diagram-wrap master-diagram-dark">
{{< inline-svg "static/images/overview/00-master-diagram-linked-dark.svg" >}}
</div>

## Chapters

{{< section-cards >}}
