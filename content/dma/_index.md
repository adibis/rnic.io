---
title: "DMA Engine"
weight: 9
description: "Every resolved key and every accepted packet eventually reaches the same place: the engine that actually moves bytes into or out of host memory. This part covers address translation, the scatter-gather descriptor walk, ordering guarantees, and outstanding-DMA tracking."
icon: switch-horizontal
cascade:
  type: docs
---

Nearly every chapter in this book eventually hands off to the same hardware: the DMA Engine is where a resolved [Key Cache](/mr/key-caching) translation and an accepted [RX PSN Check](/rx/psn-check-and-out-of-order-arrival) result both actually become bytes moving into or out of host memory. This part covers that engine directly -- address translation, the scatter-gather descriptor walk a multi-page transfer requires, the ordering guarantees software is entitled to assume, and how outstanding, in-flight DMA operations are tracked.

Chapters for this part haven't been written yet.
