---
title: "The Operation Catalog"
weight: 4
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "Every Reliable Connection opcode, First/Middle/Last/Only variant included, with the one thing each family actually asks new of the hardware: a Receive Request here, an in-place memory write there, a read-modify-write somewhere else entirely."
prev: /overview/rocev1-to-rocev2
next: /overview/one-transaction-start-to-finish
---

An opcode byte in the BTH selects one of a fixed set of operations, and this book scopes its first pass to Reliable Connection (RC) transport specifically, since that's the transport every later Part's QP is running in RTS. RC's opcode set splits into four families — SEND, RDMA WRITE, RDMA READ, and Atomics — and every multi-packet family further splits into FIRST/MIDDLE/LAST/ONLY variants: a message that fits in one packet uses the ONLY opcode, a message spanning several packets opens with FIRST, continues with as many MIDDLE packets as needed, and closes with LAST. What follows is the full table, each opcode's extended headers, and — because a header list alone doesn't say what a NIC actually has to build for it — the one hardware consequence each family adds that the others don't.

## SEND family

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

SEND is RDMA's two-sided operation: every SEND message, in any variant, consumes a Receive Request the responder posted in advance — there is no such thing as a SEND arriving with nowhere to land, by construction, because the responder's hardware is matching it against a receive queue it already provisioned. That consumption is the hardware fact that makes SEND fundamentally different from the one-sided families below: it's the one opcode family that requires the responder's software to have done work ahead of time. The invalidate variants add one more hardware action on top of normal SEND completion — revoking an rkey's validity in the responder's key table, so that any RDMA operation still in flight against that key fails cleanly instead of succeeding against a region the requester considers released. That table and its revocation path are Part 2's subject, not re-derived here.

## RDMA WRITE family

| Opcode | Extended headers | Notes |
|---|---|---|
| `RDMA_WRITE_FIRST` | BTH + RETH | opens a multi-packet WRITE, carries address/rkey/length |
| `RDMA_WRITE_MIDDLE` | BTH | continuation packet |
| `RDMA_WRITE_LAST` | BTH | closes the message |
| `RDMA_WRITE_LAST_WITH_IMMEDIATE` | BTH + IMMDT | closes the message, carries immediate data — no RETH |
| `RDMA_WRITE_ONLY` | BTH + RETH | single-packet message |
| `RDMA_WRITE_ONLY_WITH_IMMEDIATE` | BTH + RETH + IMMDT | single-packet message with immediate data |

RETH appears only on FIRST and ONLY: the destination address, rkey, and length only need stating once per message, at its start, not repeated on every packet of it. `RDMA_WRITE_LAST_WITH_IMMEDIATE` is worth calling out by name because it's an easy place to guess wrong — it does not carry RETH, matching plain `RDMA_WRITE_LAST` rather than the ONLY variants; the address information was already established back on FIRST and doesn't need restating just because this packet happens to carry immediate data too.

RDMA WRITE is one-sided: the responder's hardware performs the memory write directly against the address RETH named, with no Receive Request consumed and no completion generated on the responder side — the responder's software may never even know a plain WRITE happened, which is precisely the point of a one-sided operation. The WITH_IMMEDIATE variants break that quietly-one-sided property on purpose: immediate data has to surface to the application somehow, so those variants do consume a Receive Request and do generate a completion, turning an otherwise-invisible WRITE into something the responder's software is explicitly notified about.

## RDMA READ family

| Opcode | Extended headers | Notes |
|---|---|---|
| `RDMA_READ_REQUEST` | BTH + RETH | requester → responder: what to read |
| `RDMA_READ_RESPONSE_FIRST` | BTH + AETH | opens a multi-packet response |
| `RDMA_READ_RESPONSE_MIDDLE` | BTH | continuation packet |
| `RDMA_READ_RESPONSE_LAST` | BTH + AETH | closes the response |
| `RDMA_READ_RESPONSE_ONLY` | BTH + AETH | single-packet response |

RDMA READ is the one family where request and response are genuinely distinct opcodes traveling in opposite directions, and that split is doing real work. A `RDMA_READ_REQUEST` arriving at the responder doesn't just trigger a local memory access — it turns the responder into the initiator of a brand-new message, built entirely by its own hardware: pull the requested range out of the memory RETH named, and transmit it back as a `RDMA_READ_RESPONSE` sequence, under the responder's own PSN space for that response, not the requester's. Notice that every response opcode except MIDDLE carries AETH — the same extended header a plain acknowledgement uses. That's not incidental: a read response doubles as its own acknowledgement. A WRITE needs a separate, trailing ACK packet to tell the requester its operation landed; a READ's response payload arriving successfully is the acknowledgement, with no separate ACK message required on top of it. What AETH's syndrome byte actually communicates, and how a NIC acts on it, belongs to Part 5's reliability engine — the point to carry forward here is just that this family's hardware cost includes acting as a full message-initiator, not only a memory responder.

## Atomics

| Opcode | Extended headers | Notes |
|---|---|---|
| `COMPARE_SWAP` | BTH + ATMETH | conditional swap against a 64-bit remote word |
| `FETCH_ADD` | BTH + ATMETH | add to a 64-bit remote word, return the prior value |
| `ACKNOWLEDGE` | BTH + AETH | plain ack/NAK for SEND and WRITE-with-immediate messages |
| `ATOMIC_ACKNOWLEDGE` | BTH + AETH + ATMACK | atomic-specific ack, returns the pre-operation value |

Atomics have no FIRST/MIDDLE/LAST variants at all — both operations act on a single fixed 64-bit value, so there is no such thing as a multi-packet atomic message to frame.

Atomics are the one family that costs more at the responder than a plain DMA write does. `COMPARE_SWAP` and `FETCH_ADD` both require an in-hardware read-modify-write at the responder's memory: read the current 64-bit value, then either conditionally replace it (compare against ATMETH's comparand, swap only on match) or add to it, atomically with respect to any other operation touching that same word — and then hand the pre-operation value back to the requester in `ATOMIC_ACKNOWLEDGE`'s ATMACK field, so the requester knows what was actually there before its atomic executed. A WRITE places data; an Atomic reads, computes, and conditionally replaces, all as one indivisible responder-side action — a real memory-side compute operation, not just a placement, and meaningfully more hardware than the rest of this catalog needs.

## Scope note

Current RC opcode sets carry newer additions beyond what's cataloged above — `ATOMIC_WRITE` and `FLUSH` among them, oriented at persistent-memory and extended-atomics use cases. This book's first-pass catalog scopes to the classic eight-family set above deliberately, not by oversight; those extensions may get their own treatment in a future revision pass once the core mechanism this book builds is in place.

That's every operation an RC QP can carry. The next article follows one of them end to end — a single WQE, all the way to the wire and back to an acknowledged completion — and names, block by block, which later chapter in this book owns each piece of hardware it touches along the way.
