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

An opcode byte in the BTH selects one of a fixed set of operations, and this book scopes its first pass to Reliable Connection (RC) transport specifically, since that's the transport every later part's QP is running in RTS. RC's opcode set splits into four families — SEND, RDMA WRITE, RDMA READ, and Atomics — and every multi-packet family further splits into FIRST/MIDDLE/LAST/ONLY variants: a message that fits in one packet uses the ONLY opcode, a message spanning several packets opens with FIRST, continues with as many MIDDLE packets as needed, and closes with LAST. What follows is the full table, each opcode's extended headers, and — because a header list alone doesn't say what a NIC actually has to build for it — the one hardware consequence each family adds that the others don't. Every opcode-to-extended-header mapping below follows directly from the IBTA specification cited at the end of this chapter.

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

SEND is RDMA's two-sided operation: every SEND message, in any variant, consumes a Receive Request the responder posted in advance — there is no such thing as a SEND arriving with nowhere to land, by construction, because the responder's hardware is matching it against a receive queue it already provisioned. That consumption is the hardware fact that makes SEND fundamentally different from the one-sided families below: it's the one opcode family that requires the responder's software to have done work ahead of time. The invalidate variants add one more hardware action on top of normal SEND completion — revoking an rkey's validity in the responder's key table, so that any RDMA operation still in flight against that key fails cleanly instead of succeeding against a region the requester considers released. That table and its revocation path are [Part 2](/mr/) (Memory Region / Key Table)'s subject, not re-derived here.

![SEND/RECV sequence: the master's TX engine DMAs the source buffer and stamps PSN=N, the SEND(Only) packet crosses to the slave's RX engine, which compares PSN against its expected value, pops the pre-posted RQ WQE in order, DMAs the payload, posts a receive CQE, and ACKs -- closing the loop when the master's TX engine matches the PSN, disarms its retry timer, and posts a send CQE. A dashed side note marks the PSN-gap and RQ-empty branches without walking them.](/images/overview/04-send-recv-sequence.svg)

That pre-posting discipline plays out like this: the responder posts its Receive Request entirely out of band, before any packet arrives, and the only coordination at SEND-time is the RX engine popping whatever WQE is sitting at the head of the RQ ring, in strict order. Two failure branches fall directly out of that design — a PSN that arrives higher than the receiver's expected value gets NAK'd as a sequence error rather than processed, and a SEND arriving against an empty RQ (nothing pre-posted to consume) gets an RNR NAK carrying a retry-delay hint instead of silently stalling. Both are named here rather than walked in full: the retry, timeout, and RNR-backoff machinery behind them belongs to Part 5 (Reliability Engine), not this catalog.

## RDMA WRITE family

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

RDMA WRITE is one-sided: the responder's hardware performs the memory write directly against the address RETH named, with no Receive Request consumed and no completion generated on the responder side — the responder's software may never even know a plain WRITE happened, which is precisely the point of a one-sided operation. The WITH_IMMEDIATE variants break that quietly-one-sided property on purpose: immediate data has to surface to the application somehow, so those variants do consume a Receive Request and do generate a completion, turning an otherwise-invisible WRITE into something the responder's software is explicitly notified about. Silent to software is not the same as silent on the wire, though: RC's reliability contract still requires every WRITE, immediate or not, to be acknowledged back to the requester so the reliability engine can retire its replay-buffer entry and disarm its retry timer (Part 5 (Reliability Engine)) -- the responder's *application* may never see a plain WRITE, but the responder's *hardware* still owes the requester's hardware an ACK.

## RDMA READ family

| Opcode | Extended headers | Notes |
|---|---|---|
| `RDMA_READ_REQUEST` | BTH + RETH | requester → responder: what to read |
| `RDMA_READ_RESPONSE_FIRST` | BTH | opens a multi-packet response |
| `RDMA_READ_RESPONSE_MIDDLE` | BTH | continuation packet |
| `RDMA_READ_RESPONSE_LAST` | BTH + AETH | closes the response |
| `RDMA_READ_RESPONSE_ONLY` | BTH + AETH | single-packet response |

RDMA READ is the one family where request and response are structurally distinct opcodes traveling in opposite directions, and that split does real work. A `RDMA_READ_REQUEST` arriving at the responder doesn't just trigger a local memory access — it turns the responder into the initiator of a brand-new message, built entirely by its own hardware: pull the requested range out of the memory RETH named, and transmit it back as a `RDMA_READ_RESPONSE` sequence -- carrying the *requester's* PSN space, continued from the original request, not a PSN of the responder's own choosing. Notice that only LAST and ONLY carry AETH — the same extended header a plain acknowledgement uses; FIRST and MIDDLE carry neither, since the response isn't complete yet and there's nothing to acknowledge until it is. That's not incidental: a read response doubles as its own acknowledgement. A WRITE needs a separate, trailing ACK packet to tell the requester its operation landed; a READ's response payload arriving successfully is the acknowledgement, with no separate ACK message required on top of it. What AETH's syndrome byte actually communicates, and how a NIC acts on it, belongs to Part 5's reliability engine — the point to carry forward here is just that this family's hardware cost includes acting as a full message-initiator, not only a memory responder, and that reusing the requester's own PSN space is exactly what lets the requester's reliability engine match an incoming response back to the specific outstanding request it belongs to.

## Atomics

| Opcode | Extended headers | Notes |
|---|---|---|
| `COMPARE_SWAP` | BTH + ATMETH | conditional swap against a 64-bit remote word |
| `FETCH_ADD` | BTH + ATMETH | add to a 64-bit remote word, return the prior value |
| `ACKNOWLEDGE` | BTH + AETH | plain ack/NAK for any SEND or RDMA WRITE message, immediate or not |
| `ATOMIC_ACKNOWLEDGE` | BTH + AETH + ATMACK | atomic-specific ack, returns the pre-operation value |

Atomics have no FIRST/MIDDLE/LAST variants at all — both operations act on a single fixed 64-bit value, so there is no such thing as a multi-packet atomic message to frame.

Atomics are the one family that costs more at the responder than a plain DMA write does. `COMPARE_SWAP` and `FETCH_ADD` both require an in-hardware read-modify-write at the responder's memory: read the current 64-bit value, then either conditionally replace it (compare against ATMETH's comparand, swap only on match) or add to it, atomically with respect to any other operation touching that same word — and then hand the pre-operation value back to the requester in `ATOMIC_ACKNOWLEDGE`'s ATMACK field, so the requester knows what was actually there before its atomic executed. A WRITE places data; an Atomic reads, computes, and conditionally replaces, all as one indivisible responder-side action — a memory-side compute operation, not just a placement, and meaningfully more hardware than the rest of this catalog needs.

![Header composition across all four families side by side: SEND adds nothing but an optional IMMDT/IETH on its closing packet, WRITE and RDMA READ's request both carry RETH exactly where an address needs stating, RDMA READ's response adds AETH only when it's complete enough to double as its own acknowledgement, and Atomics always carry ATMETH since there's no multi-packet variant to spread it across.](/images/overview/04-opcode-header-composition.svg)

## Scope note

Current RC opcode sets carry newer additions beyond what's cataloged above — `ATOMIC_WRITE` and `FLUSH` among them, oriented at persistent-memory and extended-atomics use cases. This book's first-pass catalog scopes to the classic eight-family set above deliberately, not by oversight; those extensions may get their own treatment in a future revision pass once the core mechanism this book builds is in place.

That's every operation an RC QP can carry. The next article follows one of them end to end — a single WQE, all the way to the wire and back to an acknowledged completion — and names, block by block, which later chapter in this book owns each piece of hardware it touches along the way.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
