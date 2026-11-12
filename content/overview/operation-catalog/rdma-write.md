---
title: "RDMA WRITE"
weight: 2
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "The one-sided placement: the packet names the address and the responder's software never hears of it, unless immediate data asks for a completion. Still acknowledged on the wire either way."
prev: /overview/operation-catalog/send
next: /overview/operation-catalog/rdma-read
metaDescription: "How an RDMA WRITE works: the RETH names the remote address, the responder's software is never told, and WRITE with Immediate adds a completion."
seoTitle: "How RDMA WRITE Works"
---

| Opcode | Extended headers | Notes |
|---|---|---|
| `RDMA_WRITE_FIRST` | BTH + RETH | opens a multi-packet WRITE, carries address/rkey/length |
| `RDMA_WRITE_MIDDLE` | BTH | continuation packet |
| `RDMA_WRITE_LAST` | BTH | closes the message |
| `RDMA_WRITE_LAST_WITH_IMMEDIATE` | BTH + IMMDT | closes the message, carries immediate data — no RETH |
| `RDMA_WRITE_ONLY` | BTH + RETH | single-packet message |
| `RDMA_WRITE_ONLY_WITH_IMMEDIATE` | BTH + RETH + IMMDT | single-packet message with immediate data |

RETH appears only on FIRST and ONLY: the destination address, rkey, and length only need stating once per message, at its start, not repeated on every packet of it.

> [!CAUTION] Gotcha
> `RDMA_WRITE_LAST_WITH_IMMEDIATE` does not carry RETH — it matches plain `RDMA_WRITE_LAST`, not the ONLY variants. The address was already established back on FIRST and doesn't need restating just because this packet happens to carry immediate data too.

RDMA WRITE is one-sided: the responder's hardware performs the memory write directly against the address RETH named, with no Receive Request consumed and no completion generated on the responder side — the responder's software may never even know a plain WRITE happened, which is precisely the point of a one-sided operation. Silent to software is not the same as silent on the wire, though: RC's reliability contract still requires every WRITE to be acknowledged back to the requester so the reliability engine can retire its replay-buffer entry and disarm its retry timer ([Reliability Engine](/reliability/)) -- the responder's *application* may never see a plain WRITE, but the responder's *hardware* still owes the requester's hardware an ACK.

A message longer than the path MTU is where RETH-on-FIRST-only earns its keep. Play a 12 KiB WRITE leaving as three packets and watch what the responder has to remember between them:

{{< flow "write-multi" >}}

## Block by block

A single-packet WRITE across both hosts. Compare it with [SEND](/overview/operation-catalog/send)'s walk: RNIC B places the payload at the address the packet carries, and its Receive Queue fetch, CQE generation and event blocks never light.

{{< flow "blocks-write" >}}

## WRITE with Immediate

The WITH_IMMEDIATE variants break that quietly-one-sided property on purpose: immediate data has to surface to the application somehow, so those variants do consume a Receive Request and do generate a completion, turning an otherwise-invisible WRITE into something the responder's software is explicitly notified about. The address still comes from RETH; the Receive WQE supplies only somewhere to report.

The last scene of the WRITE walk above plays this variant: the payload still lands at the RETH address, and a Receive WQE is consumed only to report the completion. [One Transaction, Start to Finish](/overview/one-transaction-start-to-finish) follows a WRITE with Immediate through every stage of both adapters, packet by packet, and names the chapter that owns each one. [One All-Reduce, Four Hosts](/overview/one-all-reduce) shows the variant in bulk use: NCCL ends every chunk it moves with a WRITE with Immediate, so a single receive completion tells the receiver the whole chunk has landed.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
