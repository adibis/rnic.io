---
title: "Replay Buffer Capacity and Backpressure"
weight: 7
date: 2026-09-09
publishDate: 2026-09-09
draft: true
description: "Every other cache in this book answers capacity pressure by evicting something. This one can't -- evicting an unacknowledged entry doesn't cost a future re-fetch, it costs correctness. This chapter is what governs capacity instead."
prev: /reliability/multi-qp-retry-arbitration
next: /reliability/mid-transaction-reliability-reset
---

## Locator

![The master RNIC block diagram with the Replay Buffer highlighted, captioned to distinguish this chapter's admission-control question from the eviction-policy question Eviction Policy and Key Cache Eviction Policy both answered for two other resources.](/images/reliability/07-replay-buffer-capacity-and-backpressure-locator.svg)

[Eviction Policy](/qp/eviction-policy) and [Key Cache Eviction Policy](/mr/key-cache-eviction-policy) both answered the same question for two other cached resources: which entry gets sacrificed when capacity runs out. This chapter is where that question breaks, and what replaces it.

## Overview

A QP Cache entry evicted under pressure costs a future re-fetch from the QP Table, nothing more -- the evicted context was still valid, still correct, just no longer resident. A Key Cache entry evicted under pressure costs the same kind of re-fetch. Neither eviction loses anything a workload actually needed; both just move a cost from now to later. A Replay Buffer entry is not like either of them. This resource cannot answer a capacity question with a victim-selection policy the way the other two do.

> [!NOTE] Concept
> Evicting a QP Cache or Key Cache entry just defers a cost -- a future re-fetch. Evicting a Replay Buffer entry before an acknowledgment confirms delivery destroys the requester's only record of data it may still need to resend, permanently, for that specific operation. That's not a performance cost. It's a correctness violation.

## Block diagram

![A Send Queue about to post a new request compared against the Replay Buffer's currently available capacity for this QP: enough room admits the send and allocates an entry, insufficient room stalls the send at the TX pipeline rather than transmitting and risking an entry with nowhere to go, with no eviction path drawn anywhere in the diagram.](/images/reliability/07-replay-buffer-capacity-and-backpressure-block.svg)

## Deep walkthrough

**Why eviction is off the table here, restated precisely.** [Eviction Policy](/qp/eviction-policy) exists because a QP Cache entry has a backing copy in the QP Table that eviction merely stops keeping resident -- the data isn't lost, only relocated conceptually to a slower structure. A Replay Buffer entry has no equivalent backing copy of its own retransmission-readiness: [The Replay Buffer](/reliability/the-replay-buffer) already established that what it holds is a reference back to the original source buffer plus header state, not a duplicate that could be regenerated from some other authoritative copy if discarded. Discarding an unacknowledged entry doesn't create a cache miss recoverable by re-fetching; it creates a hole in what a requester can prove it sent, with no path back to recovering it if the peer never received it at all. LRU, priority-weighted, and error-weighted eviction all presuppose that sacrificing the chosen victim is safe. None of them are safe here.

**What governs capacity instead: never let the problem occur.** If eviction under pressure isn't survivable, the only remaining answer is preventing pressure from exceeding capacity in the first place -- admission control rather than victim selection. A Queue Pair has to be prevented from having more unacknowledged, outstanding requests than its Replay Buffer allocation can actually hold, which means the constraint has to be enforced before a new request is transmitted, not discovered after the fact when there's nowhere left to put its replay state. This is a fundamentally different kind of gate than anything else in this book's caching chapters: not "evict something to make room," but "don't create the condition that would require it."

**Where this gate actually sits, reasoned rather than sourced.** [The Replay Buffer](/reliability/the-replay-buffer) already named that outstanding-request count is naturally bounded by round-trip time and a Send Queue's own depth, but a Send Queue provisioned deeper than a specific QP's Replay Buffer allocation can afford would let software post more work than hardware can safely track for retransmission. The architecturally sound answer is a credit or windowing mechanism gating *transmission*, not posting -- a QP can have work queued and ready in its Send Queue while still being held back from actually going out, exactly the way [Single-QP Pipeline Stalls](/tx/single-qp-pipeline-stalls) already covers stall conditions gating one QP's send path without touching any other QP. This chapter adds a new stall cause to that list: a QP that has reached its Replay Buffer allocation's limit stalls at the transmit gate until an acknowledgment retires an entry and frees room for the next one, precisely mirroring how [QP Context](/qp/qp-context)'s `max_rd_atomic`/`max_dest_rd_atomic` pair already gates a different kind of outstanding-operation count for reads and atomics specifically. No public source confirms this is the exact mechanism a real implementation uses -- this book states it as the shape that's structurally required, not a documented fact.

**What this means for a workload that keeps hitting the limit.** A QP whose replay-buffer allocation is consistently the binding constraint isn't failing -- it's a QP whose peer's acknowledgments aren't returning fast enough to keep pace with how much new work the application wants outstanding at once, the same round-trip-time-driven pressure [Replay Buffer Caching](/reliability/replay-buffer-caching) already named as this resource's defining variable. The gate isn't a fault condition; it's ordinary flow control doing exactly what it's supposed to do, holding a fast sender back from outrunning what the reliability contract can actually promise to recover.

**What runs this check: a question this chapter can answer without a separate one.** Whether the admission check above is a simple fixed counter comparison or something a firmware-adjustable threshold tunes is architecturally a much smaller question than the eviction-engine question [Hardware-Assisted Eviction Engines](/qp/hardware-assisted-eviction-engines) and its Part 3/4 analogs asked of a real victim-selection policy. A counter-against-a-limit check has none of an eviction engine's complexity -- no candidates to rank, no policy to run, just a comparison gating a transmit decision. The likely shape is closer to fixed logic than a programmable engine, precisely because there's no meaningful policy space to make programmable: the only real design choice is what the limit itself is set to, a provisioning decision, not a runtime policy one.

**What happens next.** This chapter closed the last open capacity question this part's two resources raise under ordinary operation. What happens when a QP carrying armed timers and outstanding Replay Buffer entries resets mid-transaction -- the one scenario this chapter and every other in this part have deliberately deferred -- is [Mid-Transaction Reliability Reset](/reliability/mid-transaction-reliability-reset)'s subject, closing this part.

> [!TIP] Why
> Backpressure trades throughput for safety in a way eviction never had to: a QP held back from transmitting new work loses real send bandwidth for as long as the gate holds, where an evicted cache entry only ever cost a future re-fetch's worth of latency. That's a strictly worse cost on paper, and Reliable Connection pays it anyway, because the alternative -- transmitting past the point where an entry can be safely held for possible retransmission -- doesn't trade a small cost for a larger one, it trades a small cost for an unrecoverable one. A design that instead allowed transmission to outrun replay capacity, and simply gave up on retransmitting whatever overflowed, would not be offering Reliable Connection's contract at all; it would be offering something weaker while still calling itself reliable. The gate exists because the alternative isn't a worse version of this resource -- it's a different, lesser guarantee wearing this one's name.

## References

- No IETF or IBTA standard governs Replay Buffer provisioning or the specific mechanism gating transmission against it -- it is implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is reasoned architectural inference, stated as such, not a citation.
