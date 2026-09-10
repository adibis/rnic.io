---
title: "Overview"
weight: 1
description: "What RDMA demands of a NIC, RoCEv2's wire format at overview depth, and one transaction walked start to finish -- the map this book's block-by-block parts fill in afterward."
icon: book-open
cascade:
  type: docs
---

A conventional NIC moves frames between a wire and a ring buffer and stops there. An RDMA-capable NIC takes a virtual address and a remote key and moves bytes into another machine's memory directly -- no syscall, no remote CPU, no unauthorized byte moved. This part names that shift, lays out the RoCEv2 wire format that carries it, and walks one operation through the whole machine. Every stage that operation touches gets its own part later in the book.

## Chapters

{{< section-cards >}}
