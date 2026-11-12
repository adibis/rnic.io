---
title: "Queue Pair Management"
weight: 2
description: "Every RDMA operation runs against a Queue Pair, and every Queue Pair is a piece of hardware state that has to be stored, cached, installed, driven through a state machine, isolated from its neighbors when it faults, and torn down safely mid-transaction."
icon: view-grid
cascade:
  type: docs
metaDescription: "How an RDMA NIC manages Queue Pairs: the QP Table and context, caching, the state machine, error handling and isolation, and reset mid-transaction."
seoTitle: "RDMA Queue Pair Management"
homeSummary: "The QP Table and context, the QP Cache, the state machine, and what happens when a QP faults."
---

A Queue Pair is the connection a Reliable Connection RDMA operation runs on, and everything else in this book eventually answers to one: which QP a packet belongs to, what state that QP is in, and what happens when it faults. This part covers the QP end to end -- its context field by field, the table that stores it, the on-chip cache that makes touching it cheap, the software handoff that installs one, the state machine hardware enforces around it, what happens when one QP faults alone, when many fault together, when the cache runs out of room, when a QP resets with an operation still in flight, and, closing it out, how many QPs a real chip can actually hold at once and what it costs in silicon to let them share this same structure safely.

![One Queue Pair, start to finish -- what QP-owned state actually does across one operation, from creation through steady-state caching to the hazards that can interrupt it (eviction, faults, reset), closing on how many QPs can exist at once. Each step names the chapter that covers it.](/images/qp/00-qp-part-appetizer.svg)

## Chapters

{{< section-cards >}}
