---
title: "The Operation Catalog"
weight: 4
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "The four Reliable Connection operation families, the one thing each asks new of the hardware, and every operation played twice: packet by packet between requester and responder, and block by block across both hosts and both adapters."
prev: /overview/rocev1-to-rocev2
next: /overview/operation-catalog/send
metaDescription: "The four RDMA Reliable Connection operation families, SEND, WRITE, READ and atomics, each played packet by packet and block by block across two hosts."
seoTitle: "RDMA Operations: SEND, WRITE, READ, Atomics"
---

An opcode byte in the BTH selects one of a fixed set of operations, and this book scopes itself to Reliable Connection (RC) transport specifically, since that's the transport every later part's QP is running in RTS. RC's opcode set splits into four families — SEND, RDMA WRITE, RDMA READ, and Atomics — and every multi-packet family further splits into FIRST/MIDDLE/LAST/ONLY variants: a message that fits in one packet uses the ONLY opcode, a message spanning several packets opens with FIRST, continues with as many MIDDLE packets as needed, and closes with LAST. Each family has its own page below, with its full opcode table, each opcode's extended headers, and — because a header list alone doesn't say what a NIC actually has to build for it — the one hardware consequence that family adds that the others don't. Every opcode-to-extended-header mapping follows directly from the IBTA specification cited at the end of each page.

## Four families, four new demands

| Family | Operations | What it asks new of the hardware |
|---|---|---|
| [SEND](/overview/operation-catalog/send) | SEND, SEND with Immediate, SEND with Invalidate | consume a Receive WQE the responder posted in advance; with Invalidate, also revoke an rkey at the responder |
| [RDMA WRITE](/overview/operation-catalog/rdma-write) | WRITE, WRITE with Immediate | place bytes at an address the packet itself names, with no responder software involved unless immediate data asks for a completion |
| [RDMA READ](/overview/operation-catalog/rdma-read) | READ | turn the responder into a sender, building the response out of its own memory |
| [Atomics](/overview/operation-catalog/atomics) | Compare and Swap, Fetch and Add | an indivisible read-modify-write on one remote 64-bit word |

## Two ways to play each operation

Every operation on these pages can be played twice. The timing diagram puts the requester and the responder side by side and shows each packet, its headers, and what each side does when it arrives. The block diagram shows both hosts and both adapters at once: which block on which side does the work at each step, which arrows cross PCIe, and which steps run in parallel. It is the same block diagram on every page, so the difference between two operations is which blocks light up.

![Header composition across all four families side by side: SEND adds nothing but an optional IMMDT/IETH on its closing packet, WRITE and RDMA READ's request both carry RETH exactly where an address needs stating, RDMA READ's response adds AETH on every packet but MIDDLE, since it doubles as its own acknowledgement, and Atomics always carry ATMETH since there's no multi-packet variant to spread it across.](/images/overview/04-opcode-header-composition.svg)

## Scope note

Current RC opcode sets carry newer additions beyond these four families — `ATOMIC_WRITE` and `FLUSH` among them, aimed at persistent-memory and extended-atomics use cases. This catalog deliberately stops at the four classic families. FLUSH does come back in [Ordering and Outstanding DMA Tracking](/dma/ordering-and-outstanding-dma-tracking/), where it closes the gap between a completed write and a visible one.

Those four families are what the rest of this book builds on. Start with [SEND](/overview/operation-catalog/send). After Atomics, [One Transaction, Start to Finish](/overview/one-transaction-start-to-finish) follows one operation end to end and names, block by block, which later chapter owns each piece of hardware it touches.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
