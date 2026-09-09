---
title: "Queue Pair Management"
weight: 2
description: "Every RDMA operation runs against a Queue Pair, and every Queue Pair is a piece of hardware state that has to be stored, cached, installed, driven through a state machine, isolated from its neighbors when it faults, and torn down safely mid-transaction. This Part is the full ten-item checklist run once, in depth, against the QP."
icon: layout-grid
cascade:
  type: docs
---

A Queue Pair is the connection a Reliable Connection RDMA operation runs on, and everything this book's other Parts build eventually has to answer to one: which QP a packet belongs to, what state that QP is in, and what happens when it faults. This Part covers the QP end to end -- its context field by field, the table that stores it, the on-chip cache that makes touching it cheap, the software handoff that installs one, the state machine hardware enforces around it, and what happens when one QP faults alone, when many fault together, when the cache runs out of room, and when a QP resets with an operation still in flight.

[How to Read This Guide](/overview/how-to-read-this-guide) names the ten-question checklist this Part runs in full -- Part 1 is the fullest worked example of it in the book, and the template every later Part's version of the same checklist points back to.

## Chapters

{{< section-cards >}}
