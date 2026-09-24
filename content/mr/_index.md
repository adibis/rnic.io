---
title: "Memory Region / Key Table"
weight: 3
description: "Every RDMA operation carries a key instead of a QP number, and that key has to resolve to a real region, real permissions, and a real address range before a single byte moves. This part covers the Memory Region and its Key Table: descriptor, storage, caching, registration, permission checks, and eviction."
icon: key
cascade:
  type: docs
---

A remote RDMA operation doesn't address memory the way a local pointer does -- it carries an `rkey`, opaque to the requester, that the target RNIC has to resolve against its own Memory Region table before anything moves. This part covers that resolution end to end: the Memory Region descriptor a registration produces, the key table that stores it, the on-chip cache that makes resolving a key cheap, the registration handoff that installs one, what happens when a resolved key's permissions or bounds don't match the operation asking to use it, what changes when several Queue Pairs share one region, how the key cache decides what to evict under pressure, what actually executes that decision, what happens when a region is deregistered while an operation against it is still in flight, and, closing it out, how many regions a real chip can actually hold registered at once and what it costs in silicon to let registration and per-DMA lookups share the same structure safely.

Nine chapters here, not ten -- a deliberate deviation from [Queue Pair Management](/qp/)'s shape, not a gap. A QP's install-then-run lifecycle earns its own dedicated state-machine chapter because RESET→INIT→RTR→RTS is genuinely rich enough to need one; a Memory Region's own lifecycle -- registered, valid, deregistered -- doesn't have that richness, so the two questions Part 1 answers in separate chapters (installation, and the state machine governing it) collapse into one here, [Registering a Memory Region](/mr/registering-a-memory-region), without skipping either question.

![One Memory Region, start to finish -- what MR-owned state actually does across one operation, from registration through steady-state key caching to the hazards that can interrupt it (permission faults, shared arbitration, eviction, mid-transaction deregistration), closing on how many regions can exist at once. Each step names the chapter that covers it.](/images/mr/00-mr-part-appetizer.svg)

## Chapters

{{< section-cards >}}
