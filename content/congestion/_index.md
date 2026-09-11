---
title: "Congestion Control Engine"
weight: 7
description: "DCQCN rate-limiter state, tracked per QP -- another table, another working set, another eviction question, applied to a fourth hardware resource this book hasn't opened yet."
icon: adjustments
cascade:
  type: docs
---

Every Queue Pair that sends traffic carries live congestion-control state alongside it: a current injection rate, and the ECN-marked feedback that rate reacts to. This part opens that state up -- the table it lives in, the working set that makes touching it on every packet affordable, and the same eviction questions [Queue Pair Management](/qp/) and [Reliability Engine](/reliability/) already asked of a QP context and a replay buffer, asked here of a fourth resource.

Chapters for this part haven't been written yet.
