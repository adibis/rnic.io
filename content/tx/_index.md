---
title: "TX Pipeline"
weight: 4
description: "A Work Request posted to a Queue Pair doesn't become a wire packet by magic -- it's fetched off a ring, built into a real BTH/RETH/AETH packet, and arbitrated onto the wire alongside every other QP sharing the same pipeline. This part is the first one that isn't a resource with a table and a cache -- it's the datapath itself."
icon: paper-airplane
cascade:
  type: docs
metaDescription: "The transmit pipeline of an RDMA NIC: the Send Queue ring and WQE, prefetch, packet build, scheduling across QPs, and response generation."
seoTitle: "The RDMA NIC TX Pipeline"
homeSummary: "The Send Queue ring, WQE prefetch, packet build, scheduling across QPs, and the responses nobody posted."
---

Every RDMA operation described so far in this book assumed a WQE somehow becomes bytes on the wire. This part opens up that assumption: a Work Queue Element sitting in a ring in host memory, fetched by hardware, prefetched ahead of when it's actually needed, built into a real BTH/RETH/AETH packet with a valid ICRC, and arbitrated onto a pipeline shared by every other QP with traffic ready to send at the same instant -- plus what happens when one QP's send path stalls without taking any other QP down with it, and what happens when a QP resets with a WQE already mid-flight through one of these stages.

TX Pipeline has no state machine of its own -- it runs under the gate the QP's own state machine ([QP State Machine](/qp/qp-state-machine)) already provides. Installing a Send Queue's ring happens as part of [Installing a QP](/qp/installing-a-qp), not as a separate handoff. What this part covers instead: the WQE ring and its on-chip prefetch buffer, the scheduler that arbitrates many QPs onto one shared pipeline, and TX Packet Build -- the stage that turns a fetched WQE into the wire format [What Is RoCEv2](/overview/what-is-rocev2) already described field by field.

![One WQE, from post to wire -- what the TX Pipeline does to one Work Queue Element, from posting through the ring, prefetch, and packet build, to the hazards that can interrupt it (single-QP stalls, multi-QP scheduling, mid-transaction reset), plus the responses the same packet-build stage produces for the other side. Each step names the chapter that covers it.](/images/tx/00-tx-part-appetizer.svg)

## Chapters

{{< section-cards >}}
