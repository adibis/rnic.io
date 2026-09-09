---
title: "Single-QP RX Stalls"
weight: 5
date: 2026-09-09
publishDate: 2026-09-09
draft: true
description: "An arriving packet can pass every check this Part has covered and still not be processable -- not because anything is wrong with it, but because a resource it needs isn't there yet. This chapter is what that actually does to one Queue Pair's receive path, and what it does not do to any other."
prev: /rx/multi-packet-reassembly
next: /rx/multi-qp-rx-arbitration
---

## Locator

![The master RNIC block diagram with RX Parser / Reassembly and RX PSN Check both highlighted -- this chapter covers what stalls one QP's path through either stage, before the harder multi-QP question next chapter takes up.](/images/rx/05-single-qp-rx-stalls-locator.svg)

[Multi-Packet Reassembly](/rx/multi-packet-reassembly) covered what a QP's receive path needs to track across packets when everything goes right. This chapter is what happens when a packet is fully valid, correctly sequenced, and still cannot finish being processed -- for exactly one Queue Pair, in isolation.

## Overview

A packet reaching this Part's later stages has already cleared parsing and the sequence-number gate -- there is nothing wrong with the packet itself. It can still stall, because processing it to completion depends on a resource that is not available at that instant: a Receive Queue with nothing posted to it, or a DMA engine already occupied placing something else. Neither condition is a fault the way a permission violation or a bad sequence number is; both are a real operation waiting on a real, finite resource, and this chapter is what that waiting actually does to the one Queue Pair it happens to.

## Block diagram

![Two independent stall conditions feeding one Queue Pair's receive path: an empty Receive Queue triggering an RNR condition with a suggested retry delay signaled back to the sender, and a DMA engine already occupied placing a different operation's payload, both shown terminating at this QP's own progress without any arrow reaching a second QP's RX Parser / Reassembly or PSN Check state.](/images/rx/05-single-qp-rx-stalls-block.svg)

## Deep walkthrough

**The Receive Queue is empty -- a resource-not-ready condition, not a fault.** [The Operation Catalog](/overview/operation-catalog) already named this condition without opening it: a SEND-family operation arrives at a Queue Pair whose Receive Queue has nothing posted, because [The Receive Queue Descriptor and Ring](/rx/the-receive-queue-descriptor-and-ring) already established that a Receive Queue entry cannot be manufactured on demand -- it has to have been provisioned in advance. The architecturally correct response is not to drop the packet as if it were malformed, and not to force an error the way a permission violation does -- it is a signal back to the sender that the resource this operation needs is not currently available, carrying a suggested interval before the sender should try again. This is a structural hazard, not a correctness failure: nothing about the packet or the sequence is wrong, only the timing of when a receive buffer happened to be posted relative to when it was needed.

**Backpressure from the placement stage -- a different resource, the same shape of wait.** A packet that clears every check this Part runs still has to have its payload actually placed in host memory, a handoff [PSN Check and Out-of-Order Arrival](/rx/psn-check-and-out-of-order-arrival) already named without exploring further. That placement stage has finite internal capacity -- it cannot place an unbounded number of concurrent operations' payloads at once, the same scarce-resource shape [Single-QP Pipeline Stalls](/tx/single-qp-pipeline-stalls) already established for the transmit side's own finite internal queue slots. No source this book found measures this specific condition on real hardware the way that chapter's citation measured the transmit-side equivalent -- this chapter states the structural argument by direct analogy rather than inventing a number to match it: a placement stage built from finite silicon cannot serve an unbounded number of concurrent requests, and a Queue Pair whose operation arrives while that capacity is already spoken for has to wait its turn.

**What waiting does to this one Queue Pair, and only this one.** Neither condition above corrupts anything or touches any state belonging to a different Queue Pair. A Receive-Queue-empty condition delays exactly the operation that triggered it, on exactly the Queue Pair whose Receive Queue was found empty; a placement-stage backpressure condition delays exactly the packet that arrived to find the stage occupied. Nothing about either condition reaches into [Multi-Packet Reassembly](/rx/multi-packet-reassembly)'s state for a different Queue Pair, and nothing about either condition is a fault this book's error-handling chapters need to treat as an ERROR-state transition -- both resolve on their own once the resource becomes available, without requiring the Queue Pair to be torn down and rebuilt the way a real fault does.

**Where this chapter's isolation claim stops, deliberately.** This chapter has shown that a single Queue Pair's own stall does not corrupt or directly touch a different Queue Pair's own state -- but it has said nothing yet about whether many Queue Pairs' packets, each individually well-formed, competing for the same finite parsing and placement hardware, can degrade each other's *throughput* the way [Single-QP Pipeline Stalls](/tx/single-qp-pipeline-stalls) confirmed happens on the transmit side. That is a genuinely different question from the one this chapter answers, and it is [Multi-QP RX Arbitration](/rx/multi-qp-rx-arbitration)'s subject next, not this one's.

> [!TIP] Why
> Signaling a resource-not-ready condition back to the sender, rather than silently dropping the packet or forcing an immediate hard failure, is what lets a transient timing mismatch -- a Receive Queue not yet topped up, a placement stage momentarily busy -- resolve itself without tearing down a connection that has done nothing wrong. The cost of that design is entirely on the sender's side: it has to be prepared to retry, and has to wait out whatever interval the receiver suggests rather than assuming every rejection means something is broken. The alternative -- building enough Receive Queue and placement capacity that this condition could never occur -- would mean provisioning for a peak load every real deployment pays for continuously, on every Queue Pair, to avoid a condition that a bounded retry already handles correctly at a fraction of the cost.

[Multi-QP RX Arbitration](/rx/multi-qp-rx-arbitration) is next: what happens when many Queue Pairs' packets, none of them individually stalled, still compete for the same shared parsing and placement hardware.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- defines the Receive-Not-Ready condition and its retry-delay signal this chapter describes.
- No IETF or IBTA standard governs internal RX-pipeline placement-stage capacity or backpressure behavior -- it is implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines.
