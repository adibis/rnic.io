---
title: "Reliability Engine"
weight: 6
description: "A Reliable Connection promises delivery, not just transmission. This Part is the machinery that promise actually costs: what gets held until an acknowledgment confirms it's safe to forget, what happens when that acknowledgment doesn't arrive, and how long a Queue Pair is allowed to keep trying before it gives up."
icon: shield-check
cascade:
  type: docs
---

Every chapter so far that named a timer, a retry count, or an acknowledgment left the actual mechanism closed. This Part opens it: the live state a Queue Pair keeps to know what it sent and what still needs confirming, the buffer that holds an unacknowledged packet's worth of information long enough to resend it, the two parallel state machines that decide when a timeout becomes a retry and when a retry becomes a QP driven to ERROR, what actually gets retransmitted when it does, what happens when many Queue Pairs need retransmission service from the same shared pipeline at once, and what happens when a Queue Pair resets with a timer armed and a replay buffer entry still waiting on an acknowledgment that will now never come.

[How to Read This Guide](/overview/how-to-read-this-guide) named the ten-question checklist Parts 1 and 2 ran in full against a resource. This Part is a resource Part again, not a datapath the way [TX Pipeline](/tx/) and [RX Pipeline](/rx/) were, and the checklist maps onto it closely -- closer than it mapped onto either pipeline. There are two resources here, not one: per-QP retry and timeout state, and the Replay Buffer, a shared, allocable structure this book's own master-diagram planning already identified as the third full instance of the same table-plus-cache-plus-eviction pattern [QP Caching](/qp/qp-caching) and [Key Caching](/mr/key-caching) already established for two other resources. One real deviation is worth naming before it's reached rather than left for a reader to notice on their own: the Replay Buffer cannot use an LRU, priority-weighted, or error-weighted eviction policy the way a QP or key-table cache can, because evicting an unacknowledged entry early isn't a performance cost paid on a future re-fetch -- it's a correctness violation, discarding a requester's only copy of data it may still need to resend. The chapter that would have been "Eviction Policy" is instead the chapter that explains why that frame breaks here and what governs capacity in its place.

## Published

{{< section-cards >}}
