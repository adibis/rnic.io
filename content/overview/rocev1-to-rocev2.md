---
title: "RoCEv1 → RoCEv2"
weight: 3
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "RoCEv1 could only run inside one Ethernet broadcast domain. RoCEv2 fixed that by wrapping the exact same transport headers in a real IP/UDP packet — and that one change put a header-construction datapath in the NIC that RoCEv1 hardware never needed."
prev: /overview/what-is-rocev2
next: /overview/operation-catalog
---

RoCEv1 was RDMA framed directly inside Ethernet: the same Base Transport Header, extended headers, and ICRC this book has already opened up, sitting immediately after an Ethernet header carrying EtherType `0x8915`. No IP header, no UDP header, nothing routable — two RoCEv1 endpoints have to sit in the same Ethernet broadcast domain, the same L2 segment or VLAN, because there is no layer above Ethernet for a router to act on. RoCEv2 keeps every one of those transport headers unchanged and wraps them in a real IP and UDP header instead. That sounds like a small packaging change. It isn't — it's the difference between a NIC that only ever has to speak Ethernet and one that has to build and parse a full L3/L4 header on every single packet.

## RoCEv1: correct inside one broadcast domain, and nowhere else

A RoCEv1 frame has exactly one job for its Ethernet header to do: get the frame to the right MAC address on the local segment. Everything RDMA actually cares about — which QP, which PSN, which memory region — lives entirely inside the IB transport headers this book already covers, immediately following that Ethernet header. That's a genuinely minimal encapsulation, and it's also the whole problem: an Ethernet switch forwards on MAC address within a broadcast domain, and stops being able to help the moment two RDMA endpoints are separated by an IP router, which is the normal shape of anything larger than a single-switch RDMA fabric. RoCEv1 fabrics were, as a direct consequence, built as flat, single-broadcast-domain islands — workable at small scale, a hard ceiling at datacenter scale.

## RoCEv2: the same transport, a routable envelope

RoCEv2's fix doesn't touch the transport layer at all — the BTH, RETH, AETH, and every opcode this book catalogs next are byte-for-byte the same mechanism. What changes is everything around them: Ethernet header → IP header (v4 or v6) → UDP header → the unmodified IB transport headers → payload → ICRC. The UDP destination port is fixed at `4791`, IANA-assigned specifically for RoCEv2, so a receiving stack knows to hand the payload to the RDMA transport layer instead of anywhere else. The UDP source port carries no real port semantics at all — RoCEv2 doesn't listen on it — its job is to give network devices something to hash on. ECMP and LAG forwarding typically hash the standard 5-tuple, source port included, and without varying entropy there, every packet between the same pair of RDMA endpoints would hash to the same path and lose all the benefit of a multi-path fabric. Because the packet is now genuinely IP, it's genuinely routable: standard IP routing and ECMP carry it across as many hops as a normal IP flow, which is the entire point — a RoCEv2 fabric is not confined to one broadcast domain the way RoCEv1 was.

## What this actually demands of the hardware

This is the point worth stating plainly rather than skating past: RoCEv1 silicon never had to construct or parse an IP header, or a UDP header, or anything resembling either. Its per-packet header work stopped at Ethernet. RoCEv2 silicon does all of that on every packet, in both directions — on transmit, build a real IP header (source/destination address, protocol field, and for IPv4 a header checksum) and a real UDP header (fixed destination port, entropy-bearing source port, length) in front of the unchanged IB transport headers; on receive, parse and validate that same IP/UDP framing before the transport-layer parsing this book covers elsewhere even begins. That's a genuine new pipeline stage with its own datapath — header construction and, for IPv4, checksum computation on transmit; header parsing and validation on receive — not simply "a bigger header to skip over." A RoCEv1-only design could get away without ever touching an IP or UDP field. A RoCEv2 design cannot.

One more consequence of that real IP header is worth flagging here without unpacking it yet: IP headers carry ECN bits, and RoCEv2's use of a genuine IP header means those bits are available end-to-end for network devices to mark under congestion. That's the entry point DCQCN builds on — a NIC's congestion-control engine reacting to ECN marks it can only see because the packet is carrying a real IP header in the first place. Part 6 (Congestion Control Engine) covers what a NIC actually does with that signal; the point here is just that RoCEv2's routability fix is also what makes that signal exist on the wire at all.

The next article works through every operation this transport actually carries — the full opcode catalog, and what each one specifically requires the hardware to do.

## References

- InfiniBand Trade Association, *Supplement to InfiniBand Architecture Specification Volume 1, Release 1.2.1, Annex A17: RoCEv2* (2014).
- J. Postel, *Internet Protocol*, RFC 791, IETF (September 1981).
- S. Deering, R. Hinden, *Internet Protocol, Version 6 (IPv6) Specification*, RFC 8200, IETF (July 2017).
- J. Postel, *User Datagram Protocol*, RFC 768, IETF (August 1980).
- K. Ramakrishnan, S. Floyd, D. Black, *The Addition of Explicit Congestion Notification (ECN) to IP*, RFC 3168, IETF (September 2001).
