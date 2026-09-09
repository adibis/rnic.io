---
title: "RX Pipeline"
weight: 5
description: "A packet arriving at the wire is bytes, nothing more, until something takes it apart, decides whether it is even allowed to exist yet, and turns it into bytes sitting correctly in host memory. This Part is that datapath, run in the direction Part 3 built."
icon: inbox-arrow-down
cascade:
  type: docs
---

Every chapter so far that mentioned "the receiving side" left the actual mechanism unopened. This Part opens it: a frame landing on the wire, parsed apart layer by layer, checked against a sequence number before it is allowed to mean anything, tracked across however many packets one message actually spans, and finally placed in host memory -- plus what happens when one QP's receive path stalls without taking any other QP down with it, what happens when many QPs' packets arrive interleaved on the same physical pipeline, and what happens when a QP resets with a packet already mid-flight through one of these stages.

[How to Read This Guide](/overview/how-to-read-this-guide) named the ten-question checklist Parts 1 and 2 ran in full against a resource -- something installed, cached, looked up, and evicted. RX Pipeline is not that shape of thing, the same way [TX Pipeline](/tx/) was not: it is a datapath, and the checklist maps onto it imperfectly on purpose. Two items drop out entirely, for the same reasons Part 3's did: RX has no state machine of its own, only the gate the owning QP's state machine already provides, and a Receive Queue's ring is allocated as part of QP creation, not a separate handoff. One item -- caching -- maps to something structurally different again, and different from Part 3's answer too: not a demand-driven lookup, and not a lookahead buffer, but state that has to persist across every packet of one in-progress message. And the checklist misses a real transform stage twice over, not once: taking a packet apart and deciding whether to admit it are different operations, so RX gets two chapters where TX Packet Build needed only one. Eight chapters, each deviation named here rather than left for the reader to notice on their own.

## Published

{{< section-cards >}}
