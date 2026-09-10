---
title: "RX Pipeline"
weight: 5
description: "A packet arriving at the wire is bytes, nothing more, until something takes it apart, decides whether it is even allowed to exist yet, and turns it into bytes sitting correctly in host memory. This part is that datapath, run in the direction Part 3 built."
icon: inbox-in
cascade:
  type: docs
---

Every chapter so far that mentioned "the receiving side" left the actual mechanism unopened. This part opens it: a frame landing on the wire, parsed apart layer by layer, checked against a sequence number before it is allowed to mean anything, tracked across however many packets one message actually spans, and finally placed in host memory -- plus what happens when one QP's receive path stalls without taking any other QP down with it, what happens when many QPs' packets arrive interleaved on the same physical pipeline, and what happens when a QP resets with a packet already mid-flight through one of these stages.

RX Pipeline is a datapath, not a resource like the QP Table or the Key Table -- there is nothing here to install, cache, or evict. It has no state machine of its own; a receiving QP's behavior is gated by the state machine [QP State Machine](/qp/qp-state-machine) already covers. Its one piece of persistent state is per-message reassembly context, not a demand-driven cache: an address and a remaining count held open for as long as one message is in flight, then closed. And unlike [TX Packet Build](/tx/tx-packet-build), which builds a packet in a single chapter, taking a packet apart and deciding whether to admit it are different enough operations here to need two.

## Chapters

{{< section-cards >}}
