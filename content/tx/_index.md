---
title: "TX Pipeline"
weight: 3
description: "A Work Request posted to a Queue Pair doesn't become a wire packet by magic -- it's fetched off a ring, built into a real BTH/RETH/AETH packet, and arbitrated onto the wire alongside every other QP sharing the same pipeline. This Part is the first one that isn't a resource with a table and a cache -- it's the datapath itself."
icon: send
cascade:
  type: docs
---

Every RDMA operation this book has described so far assumed a WQE somehow turns into bytes on the wire. This Part is where that assumption gets opened up: a Work Queue Element sitting in a ring in host memory, fetched by hardware, prefetched ahead of when it's actually needed, built into a real BTH/RETH/AETH packet with a valid ICRC, and arbitrated onto a pipeline shared by every other QP with traffic ready to send at the same instant -- plus what happens when one QP's send path stalls without taking any other QP down with it, and what happens when a QP resets with a WQE already mid-flight through one of these stages.

[How to Read This Guide](/overview/how-to-read-this-guide) named the ten-question checklist Parts 1 and 2 both ran in full against a resource -- something installed, cached, looked up, and evicted. The TX Pipeline is not that shape of thing: it's a datapath, not a resource, and the checklist maps onto it imperfectly on purpose. Two items drop out entirely rather than being forced into a chapter that wouldn't earn its place: TX has no state machine of its own, only the gate the QP's own state machine ([QP State Machine](/qp/qp-state-machine)) already provides, and installing a Send Queue's ring is already [Installing a QP](/qp/installing-a-qp)'s job, not a separate handoff. One item -- caching -- maps to something structurally different: a prefetch buffer reading ahead of a strictly-ordered ring, not a demand-driven lookup that can hit or miss the way the QP Cache or Key Cache can. And one chapter exists entirely outside the ten-item checklist: TX Packet Build, the pipeline's actual transform stage, turning a fetched WQE into the wire format [What Is RoCEv2](/overview/what-is-rocev2) already described field by field -- a question a resource-management checklist was never built to ask, caught by checking this Part's structure against the master diagram rather than against the checklist alone. Eight chapters, not ten, each deviation named here rather than left for the reader to wonder about.

## Published

{{< section-cards >}}
