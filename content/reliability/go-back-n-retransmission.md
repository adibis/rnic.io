---
title: "Go-Back-N Retransmission"
weight: 5
date: 2026-09-09
publishDate: 2026-09-09
draft: true
description: "When a retry fires, hardware doesn't need to know exactly which packet was lost -- only where to resume from. That's cheaper to build than it sounds, and the price is wasted bandwidth on packets that technically already arrived."
prev: /reliability/the-retry-and-rnr-timeout-state-machines
next: /reliability/multi-qp-retry-arbitration
---

## Overview

[RX Packet Parse](/rx/rx-packet-parse) and [PSN Check and Out-of-Order Arrival](/rx/psn-check-and-out-of-order-arrival) already covered the responder's half of this exchange: a gap in arriving sequence numbers is rejected outright, not buffered, with a signal naming the sequence number the responder is actually waiting for. This chapter picks up from the requester's side of that same event -- what happens once a NAK naming a resume point arrives, or once a timer expires with nothing having arrived at all.

## Locator

![The master RNIC block diagram with the Replay Buffer and TX Packet Build both highlighted -- retransmission reads from the first and rebuilds through the second, without either block doing anything a normal send didn't already do.](/images/reliability/05-go-back-n-retransmission-locator.svg)

[The Retry and RNR Timeout State Machines](/reliability/the-retry-and-rnr-timeout-state-machines) named the two conditions that trigger a retry. This chapter covers what a retry -- specifically, the ordinary timeout branch's retry -- actually resends.

## Block diagram

![A requester's outstanding window with PSNs N through N+k in flight, one NAK arriving naming PSN N+2 as the resume point, and every packet from N+2 forward -- including N+3 through N+k, which may have already arrived correctly -- being rebuilt and resent, with the already-acknowledged PSNs before N+2 left untouched.](/images/reliability/05-go-back-n-retransmission-block.svg)

## Deep walkthrough

**What "go back N" actually means.** When the ordinary timeout branch fires, or when a NAK arrives naming a specific sequence number as the point the responder is waiting from, the requester's remedy is the same either way: resend every outstanding packet from that point forward, in order, not just the one packet that's suspected lost. If a NAK names PSN N+2 as the resume point, PSNs N+2 through whatever was most recently sent all get rebuilt and retransmitted -- including any of them that, unknown to the requester, actually arrived correctly the first time and are only being resent because something after them didn't.

**Why hardware doesn't need to know exactly what was lost.** [PSN Check and Out-of-Order Arrival](/rx/psn-check-and-out-of-order-arrival) already established that Reliable Connection's responder rejects an out-of-order arrival outright rather than buffering it -- which means the requester never receives a fine-grained picture of exactly which packets among a range did and didn't make it, only a single resume point. Go-back-N is the retransmission strategy that fits the information actually available: it doesn't ask the requester to reconstruct a precise loss pattern it was never told, only to trust the resume point named and resend everything after it.

> [!NOTE] Concept
> A requester rebuilding from the Replay Buffer's held state doesn't need per-packet delivery confirmation to decide what to resend -- it needs only the one PSN the responder named.

**What gets rebuilt, not what gets re-decided.** Retransmission is not a new decision about what to send -- every field a resent packet carries was already determined the first time it was sent, and [The Replay Buffer](/reliability/the-replay-buffer) held exactly what's needed to reproduce it: the header fields, and a reference back to the original Work Queue Element's source data. Rebuilding a retransmitted packet reuses [TX Packet Build](/tx/tx-packet-build)'s own mechanism -- the same header-assembly stage, working from held state instead of a freshly-fetched WQE, not a separate retransmission-specific datapath duplicating what that stage already does.

**The pure-timeout case, without a NAK.** Not every retry is triggered by an explicit rejection. A timer can expire with nothing having arrived at all -- no acknowledgment, and no NAK either, because the packet or every subsequent acknowledgment was lost outright rather than rejected. The requester's own resume point in this case is whatever [Retry and Timeout State](/reliability/retry-and-timeout-state)'s armed timer was tracking as the oldest unacknowledged PSN -- the requester doesn't need the responder to tell it where to resume, because it already knows the oldest thing it's still waiting to hear about. The retransmission itself proceeds identically either way, from the oldest unconfirmed PSN forward, whether that resume point came from an explicit NAK or from the requester's own bookkeeping.

**What happens next.** A single QP's retransmission is a self-contained event once it's triggered. What happens when several QPs need retransmission service from the same shared TX pipeline at the same time -- whether a retransmission gets any different priority than fresh traffic competing for the same resource -- is [Multi-QP Retry Arbitration](/reliability/multi-qp-retry-arbitration)'s subject next.

> [!TIP] Why
> Resending a whole range, including packets that may have already arrived correctly, trades measurable waste -- link bandwidth spent on data the peer already has -- for a much simpler requester. The alternative, selective repeat, would need the responder to communicate exactly which packets within a range it actually has, and the requester to track and resend only the specific gaps, machinery that doesn't exist anywhere else in this book's model of Reliable Connection and would need real state to track partial delivery within an outstanding window rather than a single cumulative point. Go-back-N pays in bandwidth on the rare packet that actually gets lost; selective repeat would pay in silicon complexity on every single connection, whether it ever experiences loss or not. Reliable Connection's architecture chose to spend the cost only when loss actually happens, not continuously to prepare for it.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- defines the cumulative-acknowledgment and sequence-error NAK semantics this chapter's go-back-N behavior follows from.
