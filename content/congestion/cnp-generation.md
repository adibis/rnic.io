---
title: "CNP Generation"
weight: 5
date: 2026-09-12
publishDate: 2026-09-12
draft: false
description: "Every other chapter in this Part is about the sender's own state. This one is the exception -- the receive-side mechanism that has to exist for any of that state to ever change at all, built around a real, standard-defined packet the sender never builds and the receiver never asked to send."
prev: /congestion/enabling-congestion-control
next: /congestion/the-rate-control-state-machine
---

## Overview

Nothing this Part has covered so far explains what actually triggers a rate cut. [Rate-Limiter Context](/congestion/rate-limiter-context) named the reaction factor that responds to a Congestion Notification Packet; this chapter is where that packet actually comes from. It's the one mechanism in this Part that runs on the *receive* side of a connection, triggered by something the RX pipeline sees, not anything the sending QP does.

## Locator

![The master RNIC block diagram with CNP Generation highlighted, fed by a short arrow from RX Parser / Reassembly labeled "ECN mark seen" -- every other block dimmed.](/images/congestion/05-cnp-generation-locator.svg)

Every other chapter in this Part is scoped to the sending side of a connection. This block is different: it sits on the receiving side, and what it produces travels back out to the peer that caused it.

## Block diagram

![An ECN-marked packet arriving triggers a per-flow check: has a CNP already been sent for this flow in the last N microseconds? If yes, wait. If no, build and send a CNP immediately, then arm a per-flow timer so at most one more CNP goes out for this same flow before the window closes, even if more marked packets keep arriving. The CNP itself is a real BTH-framed packet -- opcode 0x81, DestQP set to the flow being throttled, PSN forced to 0 -- built and sent the same way any other packet leaves the adapter, just triggered from the receive side instead of a posted Work Request.](/images/congestion/05-cnp-generation-block.svg)

## Deep walkthrough

**What actually triggers this.** [RX Pipeline](/rx/) already covers parsing an arriving packet's IP header; this chapter picks up from there with one additional fact that pipeline didn't need to mention until now: that IP header can carry an ECN mark, stamped by a switch somewhere along the path under load, exactly as [RoCEv1 → RoCEv2](/overview/rocev1-to-rocev2) already established. Seeing that mark on an arriving packet is what starts everything in this chapter -- nothing else about the packet changes because of it; it's still parsed, still checked, still placed the way [RX Pipeline](/rx/) already describes.

**The rate-limited notification rule -- sourced directly, not this book's own design.** A NIC doesn't send a CNP for every single marked packet it sees; that would flood the reverse path under sustained congestion. The rule, quoted directly from the algorithm this book sources throughout this Part: "If a marked packet arrives for a flow, and no CNP has been sent for the flow in last N microseconds, a CNP is sent immediately. Then, at most one CNP packet every N microseconds for the flow, if any packet that arrives within that time window was marked." The cited deployment uses N = 50µs; a second, independent implementation documented elsewhere in this Part uses a different default for its own related timer, further evidence this specific number is a tuned parameter, not a fixed constant.

> [!NOTE] Concept
> CNP generation isn't "one packet, one notification." It's a per-flow, rate-limited alarm: the first mark after a quiet period fires immediately, and every mark after that is suppressed until the window closes -- collapsing an arbitrarily bursty congestion signal down to a bounded rate of notifications.

**The packet itself is standard-defined, not this book's invention.** RoCEv2's own annex specifies a real CNP wire format: a BTH with `Opcode` set to `0b10000001` (`0x81`), `DestQP` set to the QP number of the flow being throttled (read from the marked packet that triggered this CNP), `PSN` forced to 0, `SE` and `M` forced to 0, and `P_KEY` copied from the marked packet's own BTH. [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor)'s opcode table never lists this value, for the same reason [The CQE](/cq/the-cqe)'s completion-opcode vocabulary needed values no Work Request ever requests: nothing about posting a Work Request ever asks for a CNP to be sent. It's generated entirely by this mechanism, not requested by software.

> [!CAUTION] Gotcha
> A CNP is a real packet on the wire, built and sent the same way any other packet leaves the adapter -- it isn't a side-channel signal or an internal-only event. It consumes the same egress path [TX Packet Build](/tx/tx-packet-build) already covers, competing for the same resources any other outgoing packet does, just triggered from the receive side instead of a scheduled WQE.

**Why generating CNPs promptly matters enough to be worth its own hardware budget.** DCQCN's own documentation is explicit that CNP generation is latency-sensitive: sending CNPs "with high priority, to avoid missing the CNP deadline, and to enable faster convergence." A real, named implementation's own generation capability is a bounded, finite resource: one shipped generation supports the required CNP rate for roughly 10-20 simultaneously congested flows, a later generation from the same vendor over 200. This isn't a hypothetical capacity question -- [Multi-Flow CNP Arbitration](/congestion/multi-flow-cnp-arbitration), later in this Part, covers what happens when more flows need a CNP at once than generation can keep up with.

> [!WARNING] Analogy
> This chapter is this Part's version of the "transform stage with no home in the usual ten-item checklist" gap [TX Packet Build](/tx/tx-packet-build), [RX Packet Parse](/rx/rx-packet-parse), and [CQE Generation](/cq/cqe-generation) each already filled for their own Parts -- a real, chapter-sized mechanism that doesn't map onto Context, Table, Caching, or any of the other recurring questions this book keeps asking.

**Where this connects.** The CNP this chapter builds travels back across the wire to the QP named in its `DestQP` field -- the *sender* whose traffic caused the congestion in the first place. What that sender's rate limiter actually does upon receiving it -- the rate cut, the reaction-factor update -- is [The Rate Control State Machine](/congestion/the-rate-control-state-machine), next. This chapter's own job ends the instant the CNP is handed off to the same egress path every other outgoing packet uses.

## Interface reference

| Field | Value | Source |
|---|---|---|
| **Opcode** | `0b10000001` (`0x81`) | IBTA Annex A17 (RoCEv2) |
| **DestQP** | QP number of the flow being throttled (from the marked packet's own source QP) | IBTA Annex A17 |
| **PSN** | forced to 0 | IBTA Annex A17 |
| **SE, M** | forced to 0 | IBTA Annex A17 |
| **P_KEY** | copied from the marked packet's own BTH | IBTA Annex A17 |
| **Generation window (`N`)** | 50µs in one documented deployment | DCQCN paper -- architect's choice, not standardized |

## References

- InfiniBand Trade Association, *Supplement to InfiniBand Architecture Specification Volume 1, Release 1.2.1, Annex A17: RoCEv2* (2014) -- the CNP wire format in full, cross-corroborated via a second independent secondary source (a named engineer's public whitepaper, itself footnoted directly to this same annex) rather than read from the annex text itself.
- Zhu, Eran, Firestone, Guo, Lipshteyn, Liron, Padhye, Raindel, Haj Yahia, Zhang, "Congestion Control for Large-Scale RDMA Deployments," SIGCOMM 2015 -- the rate-limited notification rule (quoted directly), the CNP-priority rationale, and the ConnectX-3/ConnectX-4 generation-capacity figures.
