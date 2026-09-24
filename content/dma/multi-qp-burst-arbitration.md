---
title: "Multi-QP Burst Arbitration"
weight: 8
date: 2026-09-16
publishDate: 2026-09-16
draft: false
description: "Ordering and Outstanding-DMA Tracking named the resource: a fixed number of transfers this engine can run at once, shared across every Queue Pair calling into it. This chapter is what happens the instant that capacity is already spoken for and one more QP's transfer needs a turn."
prev: /dma/single-burst-fault-handling
next: /dma/mid-transaction-burst-teardown
---

## Overview

[Single-QP RX Stalls](/rx/single-qp-rx-stalls) already named this chapter's trigger without describing it: "a DMA engine already occupied placing something else." That chapter's own framing is exactly right, and this chapter borrows it directly -- "neither condition is a fault the way a permission violation or a bad sequence number is; both are a real operation waiting on a finite resource." This chapter is what happens on the other side of that wait: several QPs' transfers queued for the same finite capacity, and something has to decide whose runs next.

## Locator

![The master RNIC block diagram with DMA Engine highlighted and every other block dimmed -- this chapter is the arbitration question for the concurrent-transfer capacity Ordering and Outstanding-DMA Tracking already named.](/images/dma/08-multi-qp-burst-arbitration-locator.svg)

## Block diagram

![Three Queue Pairs' transfer requests queued against one shared, fixed-size pool of concurrent-transfer slots, with three candidate policies -- FIFO, weighted, priority -- shown as alternative arbitration logic, and a long multi-segment burst already occupying one slot drawn wide enough to visibly delay several shorter requests queued behind it.](/images/dma/08-multi-qp-burst-arbitration-block.svg)

## Deep walkthrough

**The candidate policies -- the same shapes this book keeps finding at every shared resource.** FIFO services waiting requests in the order they arrived, simple and starvation-free by construction, but blind to how long any one request will actually occupy its slot. A weighted policy gives some QPs a larger share of the engine's attention -- at the cost every weighted scheme in this book pays, something external has to own and set those weights. Strict priority lets some traffic jump the queue outright, at the same starvation risk [Multi-QP RX Arbitration](/rx/multi-qp-rx-arbitration) and [Multi-QP Error Arbitration](/qp/multi-qp-error-arbitration) already named for their own priority-ordered queues.

**A real problem this resource has that a cache eviction question doesn't: transfer duration varies enormously.** A Rate Limiter entry or a QP Cache line is roughly the same size and roughly the same cost to service regardless of which QP owns it. A DMA transfer is not -- a single-segment RECEIVE and a 63-segment gather spanning several Memory Regions occupy this engine for wildly different lengths of time. A long transfer holding one of a small number of concurrent slots is a genuine head-of-line risk for every shorter request queued behind it, in a way none of this book's storage/cache/eviction resources ever had to reason about.

> [!CAUTION] Gotcha
> Don't assume a busy engine services requests in something close to the order they'd finish if run alone. A long-held slot blocks whatever's queued behind it regardless of how short the waiting request actually is -- the same head-of-line problem a single slow customer causes at a checkout line with only one register open, not a property specific to any policy above. Which policy actually mitigates it, if any does, is exactly this chapter's open question, not a settled fact.

**Gather and scatter: one shared pool, or two separate ones -- an open question, not settled here.** [In-Flight Transfer State](/dma/in-flight-transfer-state) already establishes that gather and scatter transfers run concurrently rather than as mutually exclusive phases. Whether they draw from one shared capacity pool or two independently-sized ones -- so that a burst of outgoing gathers can never starve incoming scatters entirely, or the reverse -- is a real architectural choice with no source this book found describing either way. Both are physically plausible; this book states the question rather than picking an answer it can't back.

> [!TIP] Why
> None of the candidate policies above is a universally correct choice, the same honest position this book has taken at every other shared-resource arbitration question. What's different here is how directly transfer-duration variance argues against the simplest option: FIFO is easy to build and provably fair by arrival order, but a policy blind to how long a request will actually run is a worse fit here than it was for RX arbitration's uniformly-sized packets, precisely because of the head-of-line risk named above.

**What executes whichever policy actually governs this gate is not a separate chapter here.** [Multi-QP RX Arbitration](/rx/multi-qp-rx-arbitration) and its own hardware-assisted follow-up treat "which policy" and "what runs it" as two chapters; this Part folds the second question into this one instead, the same provisional call [Event and Multi-CQ Arbitration](/cq/event-and-multi-cq-arbitration) already made for its own arbitration question -- fixed silicon, a programmable scheduling engine, or a mechanism pushed to firmware are the same three shapes this book has named for every other arbitration point, and nothing about this specific resource argues for a fourth.

**Where this connects.** A request that clears arbitration and gets a slot proceeds exactly as [The Gather Path](/dma/the-gather-path) or [The Scatter Path](/dma/the-scatter-path) already describes. What happens to a request that's still waiting for a slot when its own QP gets reset out from under it is [Mid-Transaction Burst Teardown](/dma/mid-transaction-burst-teardown)'s question, immediately next.

## Interface reference

| Policy | Fairness | Weakness |
|---|---|---|
| **FIFO** | services arrival order, starvation-free by construction | blind to transfer duration -- a long burst still blocks everything behind it |
| **Weighted** | some QPs get a larger share of slots | needs an external weight owner |
| **Priority** | latency-sensitive transfers jump the queue | starvation risk for deprioritized traffic |
| **Gather/scatter pool split** | open question -- shared or separate capacity, not settled by any source found | -- |

## References

- No IETF or IBTA standard governs internal multi-QP DMA arbitration policy -- it is implementation-specific silicon and firmware design, the same territory [Multi-QP RX Arbitration](/rx/multi-qp-rx-arbitration) and [Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling) already stated this honestly for their own shared resources.
