---
title: "RDMA READ"
weight: 3
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "The one family whose request and response are different opcodes travelling in opposite directions: the responder becomes a sender, and the response doubles as the acknowledgement."
prev: /overview/operation-catalog/rdma-write
next: /overview/operation-catalog/atomics
metaDescription: "How an RDMA READ works on the wire: a request with no payload, response packets on the requester's PSNs, and the response doubling as the ACK."
seoTitle: "How RDMA READ Works"
---

| Opcode | Extended headers | Notes |
|---|---|---|
| `RDMA_READ_REQUEST` | BTH + RETH | requester → responder: what to read |
| `RDMA_READ_RESPONSE_FIRST` | BTH + AETH | opens a multi-packet response |
| `RDMA_READ_RESPONSE_MIDDLE` | BTH | continuation packet |
| `RDMA_READ_RESPONSE_LAST` | BTH + AETH | closes the response |
| `RDMA_READ_RESPONSE_ONLY` | BTH + AETH | single-packet response |

RDMA READ is the one family where request and response are structurally distinct opcodes traveling in opposite directions, and that split does real work. A `RDMA_READ_REQUEST` arriving at the responder doesn't just trigger a local memory access — it turns the responder into the initiator of a brand-new message, built entirely by its own hardware: pull the requested range out of the memory RETH named, and transmit it back as a `RDMA_READ_RESPONSE` sequence -- carrying the *requester's* PSN space, continued from the original request, not a PSN of the responder's own choosing. Notice that FIRST, LAST and ONLY all carry AETH — the same extended header a plain acknowledgement uses — and only MIDDLE goes without. That's not incidental: a read response doubles as its own acknowledgement. A WRITE needs a separate, trailing ACK packet to tell the requester its operation landed; a READ's response payload arriving successfully is the acknowledgement, with no separate ACK message required on top of it. What AETH's syndrome byte actually communicates, and how a NIC acts on it, belongs to [Reliability Engine](/reliability/) — the point to carry forward here is just that this family's hardware cost includes acting as a full message-initiator, not only a memory responder, and that reusing the requester's own PSN space is exactly what lets the requester's reliability engine match an incoming response back to the specific outstanding request it belongs to.

Here is an 8 KiB READ, request and two-packet response, with the PSNs on every packet:

{{< flow "read" >}}

## Block by block

The same READ across both hosts. Every arrow that carried data in a WRITE's walk runs the other way here: RNIC B reads its host's memory, RNIC B's TX path builds the response, and RNIC A writes it into Host A's buffer. RNIC B's software never takes part.

{{< flow "blocks-read" >}}

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
