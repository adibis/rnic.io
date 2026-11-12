---
title: "SEND"
weight: 1
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "RDMA's two-sided operation: every SEND consumes a Receive WQE the responder posted in advance. With Immediate it carries 32 bits into the responder's completion; with Invalidate it also revokes an rkey at the responder."
prev: /overview/operation-catalog
next: /overview/operation-catalog/rdma-write
metaDescription: "How RDMA SEND works: each SEND consumes a Receive WQE posted in advance. Plus SEND with Immediate and SEND with Invalidate, packet by packet."
seoTitle: "How RDMA SEND and RECEIVE Work"
---

| Opcode | Extended headers | Notes |
|---|---|---|
| `SEND_FIRST` | BTH | opens a multi-packet SEND message |
| `SEND_MIDDLE` | BTH | continuation packet |
| `SEND_LAST` | BTH | closes a multi-packet SEND message |
| `SEND_LAST_WITH_IMMEDIATE` | BTH + IMMDT | closes the message, carries 32 bits of immediate data |
| `SEND_ONLY` | BTH | single-packet message |
| `SEND_ONLY_WITH_IMMEDIATE` | BTH + IMMDT | single-packet message with immediate data |
| `SEND_LAST_WITH_INVALIDATE` | BTH + IETH | closes the message, invalidates the rkey named in IETH |
| `SEND_ONLY_WITH_INVALIDATE` | BTH + IETH | single-packet message, invalidates the rkey named in IETH |

The invalidate variants exist only on LAST and ONLY, never FIRST or MIDDLE — invalidation is something that happens once, at the point a message actually completes, not something meaningful to attach to a packet in the middle of one.

SEND is RDMA's two-sided operation: every SEND message, in any variant, consumes a Receive Request the responder posted in advance — there is no such thing as a SEND arriving with nowhere to land, by construction, because the responder's hardware is matching it against a receive queue it already provisioned. That consumption is the hardware fact that makes SEND fundamentally different from the one-sided families: it's the one opcode family that requires the responder's software to have done work ahead of time.

{{< flow "send-recv" >}}

That pre-posting discipline plays out like this: the responder posts its Receive Request entirely out of band, before any packet arrives, and the only coordination at SEND-time is the RX engine popping whatever WQE is sitting at the head of the RQ ring, in strict order. Two failure branches fall directly out of that design — a PSN that arrives higher than the receiver's expected value gets NAK'd as a sequence error rather than processed, and a SEND arriving against an empty RQ (nothing pre-posted to consume) gets an RNR NAK carrying a retry-delay hint instead of silently stalling. Both are named here rather than walked in full: the retry, timeout, and RNR-backoff machinery behind them belongs to [Reliability Engine](/reliability/), not this catalog.

## Block by block

The same SEND across both hosts. The blocks to watch are the ones a WRITE never lights on RNIC B: the Receive Queue fetch before placement, and the completion written for the responder's software after it. The last scene adds immediate data, which changes what the completion carries and nothing else.

{{< flow "blocks-send" >}}

## SEND with Invalidate

A SEND with Invalidate carries one extra header, the IETH, holding an rkey that belongs to the responder. When the message completes, the responder invalidates that key, and its receive completion names the key it revoked. The usual case: the responder lent the requester a buffer under that rkey, the requester has finished with it, and the same SEND that says so also takes the access away. Without it, the responder's software would have to post its own local invalidate before reusing the buffer.

Not every key can be revoked this way. Linux's soft-RoCE driver accepts a memory window, or a memory region allocated for fast registration, and refuses an ordinary registration or one with windows still bound to it. What the revocation does to the [Key Table](/mr/the-key-table) and to an operation already in flight against that key is [Memory Region / Key Table](/mr/)'s subject.

{{< flow "send-invalidate" >}}

{{< flow "blocks-send-invalidate" >}}

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
- Linux kernel soft-RoCE driver, `drivers/infiniband/sw/rxe/` -- `rxe_resp.c` runs the invalidation after the payload is placed and before the completion, and reports the revoked key in the receive completion; `rxe_mr.c` (`rxe_invalidate_mr`) refuses a key that isn't a fast-registration MR or that still has memory windows bound.
