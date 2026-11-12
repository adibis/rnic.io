---
title: "Atomics"
weight: 4
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "Compare and Swap and Fetch and Add: one indivisible read-modify-write on a remote 64-bit word, the original value sent back, and a duplicate that must be answered without running twice."
prev: /overview/operation-catalog/rdma-read
next: /overview/one-transaction-start-to-finish
metaDescription: "RDMA atomics: Compare and Swap and Fetch and Add on a remote 64-bit word, the atomic acknowledgement, and why a duplicate must not run twice."
seoTitle: "RDMA Atomics: Compare-and-Swap, Fetch-and-Add"
---

| Opcode | Extended headers | Notes |
|---|---|---|
| `COMPARE_SWAP` | BTH + ATMETH | conditional swap against a 64-bit remote word |
| `FETCH_ADD` | BTH + ATMETH | add to a 64-bit remote word, return the prior value |
| `ATOMIC_ACKNOWLEDGE` | BTH + AETH + ATMACK | atomic-specific ack, returns the pre-operation value |

`ATOMIC_ACKNOWLEDGE` is the atomic family's own response. The plain `ACKNOWLEDGE` (BTH + AETH), the ACK or NAK for any SEND or RDMA WRITE message, belongs to no family: it is the reliability layer's response to SEND and WRITE alike.

Atomics have no FIRST/MIDDLE/LAST variants at all — both operations act on a single fixed 64-bit value, so there is no such thing as a multi-packet atomic message to frame.

Atomics are the one family that costs more at the responder than a plain DMA write does. `COMPARE_SWAP` and `FETCH_ADD` both require an in-hardware read-modify-write at the responder's memory: read the current 64-bit value, then either conditionally replace it (compare against ATMETH's comparand, swap only on match) or add to it, atomically with respect to any other operation touching that same word — and then hand the pre-operation value back to the requester in `ATOMIC_ACKNOWLEDGE`'s ATMACK field, so the requester knows what was actually there before its atomic executed. A WRITE places data; an Atomic reads, computes, and conditionally replaces, all as one indivisible responder-side action — a memory-side compute operation, not just a placement, and meaningfully more hardware than the rest of this catalog needs.

## Compare and Swap

A compare-and-swap that succeeds, with the original value coming back:

{{< flow "atomic-cmp-swap" >}}

The same operation across both hosts. The atomic unit on RNIC B is the one block no other family lights, and the read, the compare and the write happen there as a single step.

{{< flow "blocks-cmp-swap" >}}

## Fetch and Add

Fetch and Add always writes: it adds its operand to the word and returns what was there before. That makes a resent request dangerous in a way a resent WRITE never is. A WRITE placed twice leaves the same bytes; an add run twice counts twice. So the responder keeps the original value of each recent atomic, and answers a duplicate from what it kept instead of running it again.

{{< flow "atomic-fetch-add" >}}

{{< flow "blocks-fetch-add" >}}

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
- Linux kernel soft-RoCE driver, `drivers/infiniband/sw/rxe/` -- `rxe_mr.c` (`rxe_mr_do_atomic_op`) rejects an unaligned target and performs the read, compare or add, and write under one lock; `rxe_resp.c` (`atomic_reply`, `duplicate_request`) saves the original value with the request's PSN and replays it for a duplicate instead of executing again.
