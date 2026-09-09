---
title: "Memory Region / Key Table"
weight: 3
description: "Every RDMA operation carries a key instead of a QP number, and that key has to resolve to a real region, real permissions, and a real address range before a single byte moves. This Part runs Part 1's ten-item checklist a second time, against the Memory Region and its Key Table."
icon: key
cascade:
  type: docs
---

A remote RDMA operation doesn't address memory the way a local pointer does -- it carries an `rkey`, opaque to the requester, that the target RNIC has to resolve against its own Memory Region table before anything moves. This Part covers that resolution end to end: the Memory Region descriptor a registration produces, the key table that stores it, the on-chip cache that makes resolving a key cheap, the registration handoff that installs one, what happens when a resolved key's permissions or bounds don't match the operation asking to use it, what changes when several Queue Pairs share one region, how the key cache decides what to evict under pressure, what actually executes that decision, and what happens when a region is deregistered while an operation against it is still in flight.

[How to Read This Guide](/overview/how-to-read-this-guide) named the ten-question checklist Part 1 ran in full against the Queue Pair. This Part runs the same checklist against a different resource -- nine chapters rather than ten, since the Memory Region's valid/invalid lifecycle doesn't carry Part 1's five-state richness and folds into the registration chapter instead of earning a separate one -- and Parts 5 and 6 run it twice more yet, against a replay buffer and a rate limiter.

## Published

{{< section-cards >}}
