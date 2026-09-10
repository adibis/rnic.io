---
title: "Queue Pair Management"
weight: 2
description: "Every RDMA operation runs against a Queue Pair, and every Queue Pair is a piece of hardware state that has to be stored, cached, installed, driven through a state machine, isolated from its neighbors when it faults, and torn down safely mid-transaction."
icon: view-grid
cascade:
  type: docs
---

A Queue Pair is the connection a Reliable Connection RDMA operation runs on, and everything else in this book eventually answers to one: which QP a packet belongs to, what state that QP is in, and what happens when it faults. This part covers the QP end to end -- its context field by field, the table that stores it, the on-chip cache that makes touching it cheap, the software handoff that installs one, the state machine hardware enforces around it, and what happens when one QP faults alone, when many fault together, when the cache runs out of room, and when a QP resets with an operation still in flight.

## Chapters

{{< section-cards >}}
