---
title: "Congestion Control Engine"
weight: 7
description: "DCQCN rate-limiter state, tracked per QP -- a table this time built from an unusually well-sourced algorithm, a caching question this book can't confirm applies at all, and a real transform-stage mechanism running on the receive side, not the send side every other Part's version of this question has looked at."
icon: adjustments
cascade:
  type: docs
---

Every Queue Pair that sends traffic carries live congestion-control state alongside it: a current injection rate, and the ECN-marked feedback that rate reacts to. This part opens that state up field by field, the standard-defined Congestion Notification Packet that triggers a rate cut, and the three-phase recovery mechanism that runs in its absence -- plus a real surprise this book didn't expect going in: the interesting multi-instance question here isn't on the send side, the way it is everywhere else in this book, but on the receive side, where generating that same notification packet turns out to be a real, bounded, measured hardware resource.

## Chapters

{{< section-cards >}}
