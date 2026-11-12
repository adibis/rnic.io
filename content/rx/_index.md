---
title: "RX Pipeline"
weight: 5
description: "A packet arriving at the wire is bytes, nothing more, until something takes it apart, decides whether it is even allowed to exist yet, and turns it into bytes sitting correctly in host memory. This part is that datapath, run in the direction TX Pipeline built."
icon: inbox-in
cascade:
  type: docs
metaDescription: "The receive pipeline of an RDMA NIC: parsing RoCEv2 headers, the PSN check, multi-packet reassembly, the Receive Queue and arbitration across QPs."
seoTitle: "The RDMA NIC RX Pipeline"
homeSummary: "Parsing, the PSN check, multi-packet reassembly, and the Receive Queue."
---

Every chapter so far that mentioned "the receiving side" left the actual mechanism unopened. This part opens it: a frame landing on the wire, parsed apart layer by layer, checked against a sequence number before it is allowed to mean anything, tracked across however many packets one message actually spans, and finally placed in host memory -- plus what happens when one QP's receive path stalls without taking any other QP down with it, what happens when many QPs' packets arrive interleaved on the same physical pipeline, and what happens when a QP resets with a packet already mid-flight through one of these stages.

RX Pipeline is a datapath, not a resource like the QP Table or the Key Table -- there is nothing here to install, cache, or evict. It has no state machine of its own; a receiving QP's behavior is gated by the state machine [QP State Machine](/qp/qp-state-machine) already covers. Its one piece of persistent state is per-message reassembly context, not a demand-driven cache: an address and a remaining count held open for as long as one message is in flight, then closed. And unlike [TX Packet Build](/tx/tx-packet-build), which builds a packet in a single chapter, taking a packet apart and deciding whether to admit it are different enough operations here to need two.

![One packet, from wire to memory -- what the RX Pipeline does to one arriving packet, from parsing through PSN check and reassembly, to the hazards that can interrupt it (single-QP stalls, multi-QP arbitration, mid-transaction reset). Each step names the chapter that covers it.](/images/rx/00-rx-part-appetizer.svg)

## Chapters

{{< section-cards >}}
