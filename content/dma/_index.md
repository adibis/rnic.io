---
title: "DMA Engine"
weight: 9
description: "A normal NIC's DMA engine only ever writes into a buffer software already chose. This one sometimes does that, and sometimes places data at an address the wire packet itself names -- the one demand ordinary DMA hardware never had to meet. This part covers target selection, the gather and scatter paths that follow from it, ordering guarantees, and the shared engine's own contention and teardown questions."
icon: switch-horizontal
cascade:
  type: docs
---

Nearly every chapter in this book eventually hands off to the same hardware: the DMA Engine is where a resolved [Key Cache](/mr/key-caching) translation and an accepted [RX PSN Check](/rx/psn-check-and-out-of-order-arrival) result both actually become bytes moving into or out of host memory. This part covers that engine directly -- which address a transfer actually resolves against, the gather and scatter paths that move bytes once it has, the ordering guarantees software is entitled to assume (and the one it isn't), and what happens when this shared engine's own capacity is contended or the ground under a running transfer shifts mid-execution. This engine is also the deepest departure this book has taken from Queue Pair Management's own template: it has no QP-indexed table of its own, no installation step, and no state machine -- a shared, stateless-between-transfers engine every Queue Pair calls into, not another instance of the table-plus-cache-plus-eviction shape this book has drawn three times already.

## Chapters

{{< section-cards >}}
