---
title: "RX Packet Parse"
weight: 2
date: 2026-09-09
publishDate: 2026-09-09
draft: false
description: "Every check this chapter runs is ordered by cost, cheapest first -- a malformed header gets rejected before a single cycle goes toward the one check that actually costs something. This is the stage that takes a frame apart, in exactly the reverse order Part 3 built one."
prev: /rx/the-receive-queue-descriptor-and-ring
next: /rx/psn-check-and-out-of-order-arrival
---

## Overview

A parser is a decode-and-dispatch front end: it does not decide what an operation means, only what it is and where it belongs, before handing it to whatever downstream stage actually acts on it. An arriving frame is, at the moment it lands, nothing but bytes -- Ethernet framing, an IP header, a UDP header, and, if everything above it is well-formed, the Base Transport Header this book has already spent a full chapter describing. Every one of those layers has to be stripped and validated before the destination Queue Pair is even known, and the order that validation happens in is not arbitrary: a real implementation has every reason to reject cheaply before it pays for anything expensive.

> [!WARNING] Analogy
> This is the same discipline any pipelined decode stage applies to a malformed instruction -- reject cheaply, on a field-level check, before paying for anything that costs a real computation.

> [!NOTE] Concept
> Cheapest check first, most expensive check last. A parser rejects on framing, header sanity, and port match -- each a plain field comparison -- before it ever pays for ICRC, the one check that has to run over the whole assembled payload.

## Locator

![The master RNIC block diagram with RX Parser / Reassembly highlighted and every other block dimmed -- this chapter covers the stage that takes an arriving frame apart, the direct mirror of Part 3's TX Packet Build running in reverse.](/images/rx/02-rx-packet-parse-locator.svg)

[What Is RoCEv2](/overview/what-is-rocev2) already described the wire format this chapter takes apart, field by field. This chapter is not a second pass over that layout -- it is the pipeline stage that consumes it, and the specific order in which a real implementation has every reason to run its checks.

## Block diagram

![An arriving frame passing through four ordered checks -- Ethernet framing, IP header sanity, UDP destination port match, and only then ICRC validation over the assembled transport payload -- with a reject arrow branching off after each of the first three cheap checks and a separate, heavier reject arrow after the fourth, expensive one, illustrating cheapest-check-first ordering rather than a single monolithic validate-everything step.](/images/rx/02-rx-packet-parse-block.svg)

## Deep walkthrough

**Cheap checks first, expensive checks last -- an ordering principle, not just a sequence.** A malformed Ethernet frame, an IP header that fails basic sanity, or a UDP destination port that does not match this transport's fixed assignment are each cheap to detect -- a field comparison, not a computation over the packet's contents. The Invariant CRC that closes out the transport payload is the opposite: it has to be computed over most of what the packet carries, a real cost in silicon and cycles that dwarfs any single field comparison above it. A parser that checked ICRC before confirming the packet is even addressed to this transport at all would pay that cost on every stray frame that happens to reach the port, including frames with no legitimate claim on the RNIC's attention in the first place. Running the cheap, field-level checks first and reserving the expensive, whole-payload check for last is not a stylistic preference -- it is the only ordering that does not waste the most expensive check in the sequence on packets a cheaper check would already have rejected.

**What survives to the transport header, and what that header hands off.** Once Ethernet, IP, and UDP framing are confirmed well-formed and addressed to this transport, what remains is the Base Transport Header and whatever extended headers this specific opcode carries -- the exact catalog [The Operation Catalog](/overview/operation-catalog) already enumerated. Parsing that header is where this stage produces its one load-bearing output: the destination Queue Pair number, which drives the same kind of table-and-cache lookup [The QP Table](/qp/the-qp-table) and [QP Caching](/qp/qp-caching) already covered in depth, now performed from the receiving side instead of the sending one. Nothing about this stage re-derives that lookup's mechanics; it only produces the number that lookup consumes.

**ICRC -- the expensive check, paid last, over the payload the cheap checks already vetted.** Once the destination Queue Pair is known and the transport header's own fields are internally consistent, the Invariant CRC gets checked: recomputed over the same synthetic, router-mutable-fields-masked view of the whole packet [What Is RoCEv2](/overview/what-is-rocev2) already described -- IP/UDP header included, not just the transport header and payload -- and compared against the value the sender attached. This is the single most expensive check in the sequence, and by the time it runs, every cheaper opportunity to reject the packet has already had its chance -- which is exactly why it runs last rather than first. A packet that fails this check is not a packet whose destination Queue Pair was ever confirmed with confidence in the first place: the fields this stage read to determine that destination are exactly the fields the failed check says may not be trustworthy. The architecturally consistent response is a silent drop, not a reported fault charged against any specific Queue Pair -- there is no QP this stage can confidently blame for a packet whose own identifying fields just failed integrity validation.

**A malformed packet below the transport header is a different failure entirely, and belongs to no QP at all.** A frame that fails Ethernet, IP, or UDP-level sanity never reaches the point of naming a destination Queue Pair -- there is no BTH to read yet, because the layers above it failed first. This failure mode is not RDMA-specific and is not this book's subject in depth; it is ordinary Ethernet/IP receive hardware behavior, counted and dropped the same way any malformed frame is, well before anything RDMA-specific gets a chance to run. What matters for this chapter is only that it is architecturally distinct from an ICRC failure: one never identifies a QP; the other identifies one and then has to distrust that identification.

**What happens next.** A packet that survives every check above has a validated transport header, a confirmed destination Queue Pair, and payload bytes ready to be placed -- but it is not yet admitted. Whether it is actually allowed to proceed depends on one more gate this chapter deliberately does not run: comparing its sequence number against what the destination Queue Pair currently expects. [PSN Check and Out-of-Order Arrival](/rx/psn-check-and-out-of-order-arrival) is that gate, immediately next.

> [!TIP] Why
> Ordering validation from cheapest to most expensive is a general principle, not something specific to this book's RX pipeline -- any hardware stage that has to reject a mix of cheap-to-detect and expensive-to-detect faults benefits from paying for the expensive check only after every cheaper opportunity to avoid it has already passed. The alternative -- computing ICRC unconditionally over every arriving frame regardless of whether its framing, IP header, or UDP port even makes it a legitimate candidate -- would spend this stage's most costly operation on traffic a field comparison could have rejected for a fraction of the cost. The tradeoff this ordering accepts is a small amount of extra pipeline depth (four sequential checks instead of one combined pass), paid in exchange for never spending the expensive check on a packet a cheap one would already have caught.

[PSN Check and Out-of-Order Arrival](/rx/psn-check-and-out-of-order-arrival) is next: the gate that decides whether a parsed, validated packet is actually allowed to proceed.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- defines the Base Transport Header and Invariant CRC this chapter's parsing sequence terminates in.
- J. Postel, *Internet Protocol*, RFC 791, IETF (September 1981).
- J. Postel, *User Datagram Protocol*, RFC 768, IETF (August 1980).
