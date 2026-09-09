---
title: "Multi-QP RX Arbitration"
weight: 6
date: 2026-09-09
publishDate: 2026-09-09
draft: true
description: "The wire does not ask permission before interleaving packets from a dozen different Queue Pairs. One shared parser and one shared placement stage still have to service them in some order -- the same policy question Part 3 asked of the transmit side, asked here of arrivals nothing chose to interleave in the first place."
prev: /rx/single-qp-rx-stalls
next: /rx/hardware-assisted-rx-dispatch-engines
---

## Locator

![The master RNIC block diagram with RX Parser / Reassembly and QP Table / Cache both highlighted -- this chapter is the parser consulting per-QP state while deciding which of several arrived packets to service next.](/images/rx/06-multi-qp-rx-arbitration-locator.svg)

[Single-QP RX Stalls](/rx/single-qp-rx-stalls) confirmed that one Queue Pair's own wait does not touch a different Queue Pair's state. This chapter asks the question that leaves open: when packets belonging to several different Queue Pairs are all ready for this Part's shared hardware at once, something has to decide whose gets serviced next.

## Overview

A Send Queue chooses when to make a WQE ready to send; a Queue Pair's receiving side has no equivalent control over when the wire delivers it a packet. Many Queue Pairs sharing one adapter can have packets land within the same narrow window, all needing the same shared parser, the same sequence-number check, the same placement hardware -- and only one physical pipeline to run them through. This is the same policy shape [Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling) already worked through for the transmit side, and the same shape [Eviction Policy](/qp/eviction-policy) worked through before that for the QP Cache -- round-robin, weighted, priority -- asked here of arrival order on the receiving side instead of departure order on the sending one.

## Block diagram

![Packets from three Queue Pairs arriving at nearly the same instant and queuing at one shared RX Parser / Reassembly gate, with three candidate policies (round-robin, weighted round-robin, priority) shown as alternative arbitration logic deciding which arrived packet the shared pipeline actually processes next, and a dashed line showing an ACK-carrying packet reaching the front of a priority-ordered queue ahead of ordinary data traffic.](/images/rx/06-multi-qp-rx-arbitration-block.svg)

## Deep walkthrough

**The candidate policies -- the same shapes this book has already named twice.** Round-robin services whichever Queue Pairs have a packet waiting in a fixed rotation, simple and starvation-free by construction, but blind to the fact that servicing one Queue Pair's packet might cost this shared pipeline far more cycles than another's -- the same blind spot [Eviction Policy](/qp/eviction-policy) already named for pure least-recently-used cache selection. Weighted round-robin fixes that blindness by giving some Queue Pairs a larger share of arbitration turns than others, at the same cost every weighted scheme this book has covered pays: something external has to own and set those weights. Priority arbitration lets some arrived traffic jump the queue entirely -- at the same starvation risk [Multi-QP Error Arbitration](/qp/multi-qp-error-arbitration) already named for priority-weighted fault reporting, a steady stream of high-priority arrivals can, in principle, indefinitely delay whatever is sharing the same gate at lower priority.

**A real candidate for priority treatment, reasoned here rather than sourced.** An arriving acknowledgment carries no payload to place and exists purely to let the sending side retire outstanding work and advance its own progress -- unlike a data-carrying packet, servicing it quickly has a direct effect on a different Queue Pair's transmit-side throughput, not just this one's receive-side bookkeeping. This is a plausible, architecturally motivated reason a real implementation might prioritize acknowledgment traffic over ordinary data arrivals at this gate, offered as reasoned analysis rather than a documented fact -- no source this book found confirms a real implementation actually does this, the same honest limit [Single-QP RX Stalls](/rx/single-qp-rx-stalls) already hit for placement-stage backpressure.

**What is confirmed, and what is not.** A direct measurement of real adapter hardware names a transmit-side processing unit and a receive-side processing unit as separate blocks sharing common on-chip cache resources between them -- confirming, independently of this book's own reasoning, that the receiving side has dedicated hardware distinct from the transmit side rather than a single undifferentiated pipeline handling both directions. What that measurement does not do is describe the receive-side unit's own internal arbitration policy the way a separate, vendor-quoted admission confirmed an internal scheduling policy exists on the transmit side. This chapter states that gap honestly: the *existence* of dedicated receive-side hardware is confirmed; *which* policy governs arbitration within it is not.

**Why this is not simply Multi-QP TX Scheduling's argument restated.** The transmit-side chapter's central finding was that congestion control does not substitute for a real scheduling policy, because the transmit-side problem is internal to the adapter and exists even on an uncongested network. The equivalent claim on the receive side is different in kind, not just in direction: nothing about congestion control is even in scope here, because this chapter's contention is over shared parsing and placement hardware for packets that have already crossed the wire successfully -- congestion control governs whether and how fast a sender's packets reach the wire in the first place, a question that is settled by the time this chapter's arbitration even begins. This chapter's problem starts precisely where a congestion-control mechanism's job already ends.

**What happens next.** Whichever policy actually governs this gate, something has to run it -- fixed silicon, a programmable engine, or a mechanism pushed out to software entirely. That is [Hardware-Assisted RX Dispatch Engines](/rx/hardware-assisted-rx-dispatch-engines)'s subject, immediately next.

> [!TIP] Why
> None of the three policies above is a universally correct choice for this gate any more than it was for cache eviction or transmit scheduling -- round-robin is cheap and fair but blind to cost-per-packet; weighted round-robin fixes that at the cost of an external weight owner; strict priority serves genuinely time-sensitive traffic (an acknowledgment unblocking a different Queue Pair's own progress, plausibly) predictably fast, at a real starvation risk for whatever it deprioritizes. What is different here from the transmit-side version of this question is how much less public evidence exists to ground the choice: the transmit side had a vendor's own quoted admission and an independent measured study to draw from; the receive side has confirmed dedicated hardware and an honest gap about what runs inside it. This chapter states the tradeoff plainly without overclaiming which side of it any real implementation has actually chosen.

[Hardware-Assisted RX Dispatch Engines](/rx/hardware-assisted-rx-dispatch-engines) is next: what actually executes whichever arbitration policy this chapter named.

## References

- No IETF or IBTA standard governs internal multi-QP RX arbitration policy -- it is implementation-specific silicon and firmware design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above, not a standards document.
