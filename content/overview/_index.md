---
title: "Overview"
weight: 1
description: "What RDMA demands of a NIC, RoCEv2's wire format at overview depth, and one transaction walked start to finish -- the map this book's block-by-block Parts fill in afterward."
icon: book-open
cascade:
  type: docs
---

A conventional NIC moves frames between a wire and a ring buffer and stops there. An RDMA-capable NIC is handed a virtual address and a remote key and is expected to move bytes into another machine's memory without a syscall, without the remote CPU, and without a single unauthorized byte moved. This Part names that shift, lays out the concrete RoCEv2 wire format that carries it, and walks one operation through the whole machine before the rest of this book opens each stage it touched into its own Part.

## Published

{{< section-cards >}}
