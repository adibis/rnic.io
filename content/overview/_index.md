---
title: "Overview"
weight: 1
description: "What RDMA demands of a NIC, RoCEv2's wire format at overview depth, and one transaction walked start to finish -- the map this book's block-by-block parts fill in afterward."
icon: book-open
cascade:
  type: docs
metaDescription: "RDMA NIC architecture from the top: what RDMA demands of a NIC, the RoCEv2 wire format, the four operation families, and one transaction end to end."
seoTitle: "RDMA NIC Architecture Overview"
homeSummary: "What RDMA asks of a NIC, the RoCEv2 packet, the four operation families, and one transaction end to end."
---

A conventional NIC moves frames between a wire and a ring buffer and stops there. An RDMA-capable NIC takes a virtual address and a remote key and moves bytes into another machine's memory directly -- no syscall, no remote CPU, no unauthorized byte moved. This part names that shift, lays out the RoCEv2 wire format that carries it, and walks one operation through the whole machine. Every stage that operation touches gets its own part later in the book.

## The whole chip, in one picture

Below is the same diagram every chapter in this book opens with -- normally with just one box lit up to show where that chapter fits. Here it's shown whole, before anything gets zoomed in on.

The dashed boxes on the left and right sit in the host machine, not on the RNIC chip itself -- that's where an operation starts and where it eventually gets reported back. The amber boxes store or look something up (a connection's state, an address translation, an unacknowledged packet waiting to be resent). The blue boxes are where a packet actually gets built, parsed, or moved. Everything else is just the wiring between them.

Every box is clickable -- click one to go straight to its chapter. Once you've read [One Transaction, Start to Finish](/overview/one-transaction-start-to-finish), [play that transaction across this diagram](/overview/one-transaction-start-to-finish/#flow-chip-walk=1), block by block.

---

<div class="master-diagram-wrap rnic-themed-svg">
{{< inline-svg "static/images/overview/00-master-diagram-linked.svg" "The RNIC master block diagram: every hardware block between a Work Request and the wire, each one a link to the chapter that covers it" >}}
</div>

## Chapters

{{< section-cards >}}
