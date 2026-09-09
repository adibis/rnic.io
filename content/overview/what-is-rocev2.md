---
title: "What Is RoCEv2"
weight: 2
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "A RoCEv2 packet looks like an ordinary UDP datagram to everything sitting between two endpoints, and nothing like one to either end running verbs -- what's actually packed inside is where this book's build-by-block depth begins."
prev: /overview/what-is-rdma
next: /overview/rocev1-to-rocev2
---

[What Is RDMA](../what-is-rdma) named four things a normal NIC never had to do: bypass the kernel on the data path, enforce memory protection in hardware instead of the OS, run a reliability protocol in hardware instead of a software TCP stack, and let a remote adapter touch memory without waking the remote CPU. None of that says anything yet about what actually goes out on the wire. RoCEv2 — RDMA over Converged Ethernet, version 2 — is the concrete protocol this book uses throughout: the wire format that carries every one of those capabilities as real bytes, at real offsets, in a real header. This chapter stays at overview depth, naming every structure precisely enough to place Parts 3 through 8's deep dives against it. It doesn't yet explain how a receiver validates a PSN or how a destination QP gets looked up — those chapters own that.

## RoCEv2 rides inside an ordinary UDP/IP packet

A RoCEv2 packet is, from the outside, just UDP over IP over Ethernet — a deliberate choice covered in full in the next chapter. Inside that UDP payload sits the actual RDMA transport: a Base Transport Header, whatever extended headers that packet's operation requires, the payload itself, and a 4-byte integrity trailer specific to RDMA, all ahead of the Ethernet frame's own trailer.

![A RoCEv2 packet's full layer stack: Ethernet, IPv4/IPv6, and UDP forming an outer shell indistinguishable from ordinary routed traffic, followed by the actual RDMA transport -- BTH on every packet, an opcode-dependent extended header, the payload, and a 4-byte ICRC trailer -- all ahead of the Ethernet frame's own FCS.](/images/overview/02-rocev2-packet-stack.svg)

The UDP destination port is fixed at 4791 for every RoCEv2 packet, regardless of operation. The source port carries no RDMA meaning at all — it exists so routers and switches doing ECMP or LAG hashing (which typically hash the full 5-tuple, UDP ports included) see varying entropy across different flows between the same pair of adapters, instead of every packet between two QPs hashing identically and collapsing onto one path. The adapter picks it from the dynamic/ephemeral port range and varies it per flow; the exact construction is implementation-specific and not something this book states as a fixed formula.

## The Base Transport Header carries every packet's identity

Every RoCEv2 packet, whatever its operation, opens with the same 12-byte Base Transport Header (BTH):

```
 byte offset   0          1              2–3          4–7                 8–11
             +----------+--------------+------------+-------------------+-------------------+
             | OpCode   | Flags        | P_Key      | Dest QPN word     | PSN word          |
             | (8 bits) | (SE, M,      | (16 bits)  | (24-bit dest. QP  | (24-bit packet    |
             |          |  PadCnt,     |            |  number, packed   |  sequence number, |
             |          |  TVer)       |            |  with a reserved  |  packed with the  |
             |          |              |            |  bit)             |  AckReq bit)      |
             +----------+--------------+------------+-------------------+-------------------+
```

OpCode is the field everything else keys off: it names both the operation (SEND, RDMA WRITE, RDMA READ, an atomic, an acknowledgement) and, for a multi-packet message, that packet's position in it (FIRST, MIDDLE, LAST, or ONLY for a single-packet message) — the full table is 0.4's job. The destination QP number and the PSN each get a 24-bit field, packed alongside a couple of other bits (a reserved bit riding with the QPN, an ACK-request flag riding with the PSN) whose exact bit positions this overview doesn't pin down — that belongs to whichever pipeline-stage chapter actually parses the header (Part 4, RX Pipeline). Congestion signaling rides in the outer IP header instead, not in the BTH at all — [RoCEv1 → RoCEv2](../rocev1-to-rocev2) covers why that placement is exactly what routability bought this protocol. What's fixed, and carries forward from here: the destination QP number is how a receiving adapter knows which QP context to look up (Part 1.2/1.3), and the PSN is what the reliability engine (Part 5 (Reliability Engine)) checks on every packet to detect loss, duplication, and reordering.

## Extended headers ride behind the BTH, one operation at a time

The BTH alone is enough for some packets (a SEND-family MIDDLE, say). Others carry an additional, opcode-specific header immediately after it:

| Header | Carries | Size |
|---|---|---|
| **RETH** — RDMA Extended Transport Header | remote virtual address, `rkey`, DMA length | 16 bytes |
| **AETH** — ACK Extended Transport Header | an 8-bit syndrome plus a 24-bit Message Sequence Number | 4 bytes |
| **IMMDT** — Immediate Data | a 32-bit application-supplied value | 4 bytes |
| **IETH** — Invalidate Extended Transport Header | the 32-bit `rkey` to invalidate | 4 bytes |
| **ATMETH** — Atomic Extended Transport Header | remote virtual address, `rkey`, swap/add operand, compare operand | 28 bytes |

RETH is how a WRITE or READ tells the responder where to touch memory — and, notably, it only needs to appear once per message: on a multi-packet transfer's FIRST or ONLY packet, never on MIDDLE or LAST, since the address and length don't change mid-message. ATMETH is sized for the largest operation in the catalog — a 64-bit compare plus a 64-bit swap/add value alongside the usual address and key — which is also why atomics (0.4) never span multiple packets: there's no FIRST/MIDDLE/LAST variant for an operation that's already one fixed-size unit.

> [!CAUTION] Gotcha
> AETH's syndrome byte is not a generic "ack happened" flag — it's the field that distinguishes a plain ACK from an RNR NAK from every other NAK type, exactly the signal [Part 1](/qp/) (Queue Pair Management)'s error-handling chapters and Part 5's reliability engine both key off.

## The ICRC closes the packet with an integrity check that survives routing

A RoCEv2 packet ends with a 4-byte Invariant CRC (ICRC), sitting after the payload and ahead of the outer Ethernet frame's own FCS trailer. The name is the important part: because a RoCEv2 packet gets routed — the whole point of the "v2" in RoCEv2, covered next — some header bytes are legitimately allowed to change as the packet crosses hops between source and destination. An end-to-end integrity check can't be computed against those bytes' literal in-transit value, or a completely healthy, correctly-routed packet would fail it at the destination. So the ICRC is computed against a synthetic view of the packet instead: every field that's allowed to vary in transit is forced to all 1-bits before the CRC runs, both when the sender generates it and when the receiver checks it, so routing-legal changes never affect the result. Which specific fields count as variant is a fixed list defined by the protocol; enumerating that list precisely belongs to whichever chapter in the TX and RX pipeline Parts ends up owning ICRC generation and checking, not this overview.

That's every structure this book will keep coming back to: an outer UDP/IP shell whose only RDMA-relevant content is a fixed destination port, a BTH on every packet naming the operation and the destination QP, an opcode-specific extended header carrying whatever that operation actually needs, and an integrity trailer computed to survive the routing RoCEv2 was built for. [RoCEv1 → RoCEv2](../rocev1-to-rocev2) explains why that UDP/IP shell exists at all — what RoCEv1 was before it, and specifically what routability demanded of the hardware that a single-broadcast-domain protocol never had to provide.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
- InfiniBand Trade Association, *Supplement to InfiniBand Architecture Specification Volume 1, Release 1.2.1, Annex A17: RoCEv2* (2014).
- J. Postel, *Internet Protocol*, RFC 791, IETF (September 1981).
- S. Deering, R. Hinden, *Internet Protocol, Version 6 (IPv6) Specification*, RFC 8200, IETF (July 2017).
- J. Postel, *User Datagram Protocol*, RFC 768, IETF (August 1980).
- Internet Assigned Numbers Authority, *Service Name and Transport Protocol Port Number Registry* (UDP port 4791, "roce").
