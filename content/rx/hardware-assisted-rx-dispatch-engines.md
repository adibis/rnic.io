---
title: "Hardware-Assisted RX Dispatch Engines"
weight: 7
date: 2026-09-09
publishDate: 2026-09-09
draft: true
description: "The receive path has a constraint the transmit side never faces: a packet already crossed the wire before this pipeline gets a say in when it arrives. This chapter asks the same fixed-versus-programmable question Part 3 asked of transmit scheduling, and finds a real structural reason the honest answer may differ."
prev: /rx/multi-qp-rx-arbitration
next: /rx/mid-transaction-rx-reset
---

## Overview

A dispatch decision that has to run on every single arrival competing for shared parsing and placement hardware can be hardwired at design time, built as a programmable engine, or pushed out to software. [Hardware-Assisted Eviction Engines](/qp/hardware-assisted-eviction-engines) already worked through this exact spectrum in full, and that reasoning applies here without needing to be re-derived. What this chapter adds is a real structural argument for where the receive side is likely to sit on it, distinct from the transmit side's answer -- not because less thought went into receive-side hardware, but because the receive side faces a constraint transmit scheduling does not.

## Locator

![The master RNIC block diagram with RX Parser / Reassembly highlighted -- the same target as Multi-QP RX Arbitration, this chapter asks what hardware actually runs the policy that chapter named.](/images/rx/07-hardware-assisted-rx-dispatch-engines-locator.svg)

[Multi-QP RX Arbitration](/rx/multi-qp-rx-arbitration) named the candidate policies. This chapter asks what actually executes whichever one a given design picks -- the same fixed-versus-programmable-versus-software-driven question [Hardware-Assisted Eviction Engines](/qp/hardware-assisted-eviction-engines) and [Hardware-Assisted TX Scheduling Engines](/tx/hardware-assisted-tx-scheduling-engines) already asked of two other resources.

## Block diagram

![The same three-position spectrum used in the two earlier hardware-assisted chapters -- fixed-function / programmable engine / software-driven -- redrawn for RX dispatch, with the software-driven position shown faded and a callout explaining why a wire-rate arrival gate resists the software-trap pattern the transmit side's own softwaredriven example used.](/images/rx/07-hardware-assisted-rx-dispatch-engines-block.svg)

## Deep walkthrough

**The spectrum itself -- not re-derived, cross-referenced.** [Hardware-Assisted Eviction Engines](/qp/hardware-assisted-eviction-engines) already worked through the full latency-versus-flexibility tradeoff between fixed-function hardware, a programmable engine, and a software-driven trap. Every part of that reasoning -- fixed silicon is fastest and least flexible; a programmable engine is the realistic middle ground most designs reach for; a software trap on every decision reintroduces exactly the round-trip RDMA hardware offload exists to remove -- applies here as directly as it applied to transmit scheduling. This chapter does not re-argue it a second time; it applies the same spectrum to a different decision and reasons about where the receive side's own constraints are likely to place it.

**A constraint the transmit side never has to face.** [Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling) covered a decision hardware makes about work it already holds -- WQEs sitting in host memory, under no time pressure to be serviced within any fixed window, because nothing forces a Send Queue's ring to drain at any particular rate. This chapter's decision is not that shape of problem: a packet has already crossed the wire and arrived before this pipeline gets any say in the matter, and every packet's sequence-number check has to complete quickly enough that the shared parsing and placement hardware keeps pace with line rate rather than accumulating an ever-growing backlog. A dispatch decision on the receive side is under a real timing pressure the transmit-side scheduling decision simply does not share.

**Why that constraint argues against the software-driven end of the spectrum specifically.** [Hardware-Assisted TX Scheduling Engines](/tx/hardware-assisted-tx-scheduling-engines) documented a working example of transmit-side scheduling pushed entirely into software, built specifically because the hardware it targeted had no programmable packet scheduler at all. That pattern is far less architecturally attractive on the receive side: a software trap invoked on every arriving packet's dispatch decision would reintroduce a round trip on a path that has to sustain line rate, not merely process work whenever the pipeline gets around to it. This is reasoned architectural inference, not a documented fact about any specific real implementation -- no source this book found confirms or rules out a software-driven receive-side dispatch mechanism -- but the timing argument above is a structural reason to expect the receive side to lean further toward fixed or programmable hardware than the transmit side's own documented example does.

**What is and is not known about where real implementations actually sit.** This chapter has less public evidence to draw from than either of the two earlier hardware-assisted chapters. [Hardware-Assisted TX Scheduling Engines](/tx/hardware-assisted-tx-scheduling-engines) had a vendor's own quoted admission of a firmware-adjustable internal policy to anchor its middle-of-the-spectrum claim; this chapter has no equivalent citation for receive-side dispatch specifically. What is confirmed, from the same measured source [Multi-QP RX Arbitration](/rx/multi-qp-rx-arbitration) already cited, is that a real implementation dedicates distinct processing hardware to the receive side rather than routing it through the same pipeline as transmit -- consistent with, though not proof of, a design that treats receive-side dispatch as latency-sensitive enough to warrant its own hardware rather than sharing a more flexible, possibly software-adjacent mechanism with the transmit side. This chapter states that as what it is: a structurally reasoned expectation, not a sourced fact.

**What happens next.** This chapter closes the arbitration question this part opened with [Single-QP RX Stalls](/rx/single-qp-rx-stalls) and continued through [Multi-QP RX Arbitration](/rx/multi-qp-rx-arbitration). What remains is a question orthogonal to all of it: what happens to a packet caught mid-parse, mid-sequence-check, or mid-reassembly when the Queue Pair it belongs to resets underneath it.

> [!TIP] Why
> The same tension [Hardware-Assisted Eviction Engines](/qp/hardware-assisted-eviction-engines) and [Hardware-Assisted TX Scheduling Engines](/tx/hardware-assisted-tx-scheduling-engines) both named applies here with an additional real constraint layered on top: a receive-side dispatch decision has to keep pace with an arrival rate this pipeline does not control, which makes the software-driven end of the spectrum a structurally worse fit here than it was for transmit scheduling, where a working example nonetheless shows it is not impossible. A vendor building fixed or narrowly-programmable receive-side dispatch hardware accepts less runtime flexibility in exchange for a dispatch decision that cannot become the bottleneck limiting how fast this pipeline can sustain line rate -- a trade the timing pressure above makes considerably more attractive here than it was for a transmit-side decision with no equivalent deadline.

[Mid-Transaction RX Reset](/rx/mid-transaction-rx-reset) is next: what happens to a packet caught mid-parse, mid-check, or mid-reassembly when its owning Queue Pair resets underneath it.

## References

- No IETF or IBTA standard governs which hardware runs a receive-side dispatch policy -- it is implementation-specific silicon and firmware design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above and in reasoned architectural inference stated as such, not a standards document.
