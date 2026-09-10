---
title: "Memory Region / Key Table"
weight: 3
description: "Every RDMA operation carries a key instead of a QP number, and that key has to resolve to a real region, real permissions, and a real address range before a single byte moves. This part covers the Memory Region and its Key Table: descriptor, storage, caching, registration, permission checks, and eviction."
icon: key
cascade:
  type: docs
---

A remote RDMA operation doesn't address memory the way a local pointer does -- it carries an `rkey`, opaque to the requester, that the target RNIC has to resolve against its own Memory Region table before anything moves. This part covers that resolution end to end: the Memory Region descriptor a registration produces, the key table that stores it, the on-chip cache that makes resolving a key cheap, the registration handoff that installs one, what happens when a resolved key's permissions or bounds don't match the operation asking to use it, what changes when several Queue Pairs share one region, how the key cache decides what to evict under pressure, what actually executes that decision, and what happens when a region is deregistered while an operation against it is still in flight.

## Chapters

{{< section-cards >}}
