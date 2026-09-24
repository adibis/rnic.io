---
title: "Reliability Engine"
weight: 6
description: "A Reliable Connection promises delivery, not just transmission. This part covers what keeping that promise actually costs: what gets held until an acknowledgment confirms it's safe to forget, what happens when that acknowledgment doesn't arrive, and how long a Queue Pair is allowed to keep trying before it gives up."
icon: shield-check
cascade:
  type: docs
---

Every chapter so far that named a timer, a retry count, or an acknowledgment left the actual mechanism closed. This part opens it: the live state a Queue Pair keeps to know what it sent and what still needs confirming, the buffer that holds an unacknowledged packet's worth of information long enough to resend it, the two parallel state machines that decide when a timeout becomes a retry and when a retry becomes a QP driven to ERROR, what actually gets retransmitted when it does, what happens when many Queue Pairs need retransmission service from the same shared pipeline at once, and what happens when a Queue Pair resets with a timer armed and a replay buffer entry still waiting on an acknowledgment that will now never come.

This part covers two resources, not one: per-QP retry and timeout state, and the Replay Buffer itself.

One difference matters before you reach it. The Replay Buffer can't use an LRU, priority-weighted, or error-weighted eviction policy the way a QP or key-table cache can: evicting an unacknowledged entry early isn't a performance cost paid on a future re-fetch, it's a correctness violation -- discarding a requester's only copy of data it may still need to resend. The chapter that would have been "Eviction Policy" instead explains why that frame breaks here, and what governs capacity in its place.

![One sent packet, until it's safe to forget -- what reliability-owned state does after a packet leaves, from the held copy and armed timer through the retry state machine, to the hazards that can interrupt it (multi-QP arbitration, backpressure, mid-transaction reset). Each step names the chapter that covers it.](/images/reliability/00-reliability-part-appetizer.svg)

## Chapters

{{< section-cards >}}
