---
title: "Message Completion and the MSN"
weight: 3
date: 2026-09-18
publishDate: 2026-09-18
draft: false
description: "PSN answers which packet arrived. It cannot answer which Work Request is done, because software posts work at message granularity and a single message can span many packets. This chapter covers the field that answers that question instead, and the one handoff no earlier chapter in this Part actually owns."
prev: /reliability/the-replay-buffer
next: /reliability/replay-buffer-caching
metaDescription: "How the AETH's Message Sequence Number tells an RDMA requester which Work Requests are done when one message spans many packets."
seoTitle: "RDMA Message Completion and the MSN"
---

## Overview

[Retry and Timeout State](/reliability/retry-and-timeout-state) and [The Replay Buffer](/reliability/the-replay-buffer) both track what an acknowledgment retires by Packet Sequence Number -- a strictly per-packet count. That leaves a real question unanswered: software posts Work Requests at message granularity, not packet granularity, and a single message (a large WRITE, a multi-packet READ) can span many PSNs while corresponding to exactly one unit of completed work. The Message Sequence Number is the AETH's other field, carried alongside Syndrome in every acknowledgment, and it exists specifically to answer that question -- not "which packet did you get" but "which message have you finished." This chapter defines it, and closes the one handoff nothing earlier in this Part owns: how a completed message actually becomes a completion entry in [Completion Queue / Event Management](/cq/).

## Locator

![The master RNIC block diagram with the Replay Buffer / Retry Timer block and CQE Gen both highlighted, and the arrow between them lit -- the path an inbound acknowledgment actually takes to become a local completion.](/images/reliability/03-message-completion-and-the-msn-locator.svg)

[The Replay Buffer](/reliability/the-replay-buffer) covered what an acknowledgment retires by Packet Sequence Number. This chapter covers what that same acknowledgment also carries -- a second, message-granularity count -- and what it triggers once retirement is done.

## Block diagram

![An inbound ACK's AETH split into an 8-bit Syndrome and a 24-bit MSN; the requester's own per-QP Work Queue Element list walked from the oldest outstanding entry forward, each WQE's logical Send Sequence Number compared against the new MSN and retired if covered -- stopping early at the first outstanding RDMA READ or ATOMIC request even if the MSN value covers WQEs behind it -- with a retired WQE handed to CQE Generation as a local, this-side-only completion.](/images/reliability/03-message-completion-and-the-msn-block.svg)

## Deep walkthrough

**What MSN counts, and why PSN alone can't answer this.** The AETH (4 bytes) carries two fields: an 8-bit Syndrome and a 24-bit MSN. IBTA-V1 §9.3.5.2 defines it directly: "monotonically increasing (modulo 2^24) sequence number of the last message completed at the responder. This field is used to optimize completion processing at the requester." Where PSN is scoped to one packet, MSN is scoped to one *message* -- one posted Work Request, regardless of how many packets carrying it took to cross the wire. AETH itself is carried on every ACK, and on the first and last packet of an RDMA READ Response specifically (§9.3.5) -- not on every packet of a multi-packet READ response, only the two that bracket it.

**Generated once per completed message, never for a duplicate.** IBTA-V1 §9.7.7.1's compliance statements (C9-147, C9-148) fix the generation rule for Reliable Connection service: the responder initializes MSN to zero and increments it whenever it completes a new, valid request message -- explicitly **not** for a duplicate request the reliability layer happened to retransmit and the responder re-received. For an ordinary SEND or WRITE, the incremented value goes out in that message's own response. For an RDMA READ or ATOMIC specifically, the responder has a choice: increment early, right after validating the request and before transmitting any data, returning the already-incremented value in the *first* response packet -- or increment late and return it in the *last* response packet instead. Either placement is compliant; the increment itself happens exactly once per request message, either way. C9-149 adds one more guarantee: MSN is inserted in the AETH "regardless of whether the response is a positive acknowledgment, a negative acknowledgment, or a duplicate acknowledgment" -- it's always present and valid whenever AETH is, independent of what kind of packet is carrying it.

**The requester's retirement rule -- and how one ACK can close out several Work Requests at once.** IBTA-V1 §9.7.7.1.1 introduces a purely logical Send Sequence Number: a sequential number the requester conceptually assigns each WQE it posts, in a stated one-to-one relationship with the MSN the responder will eventually return for that message. ("This... is a logical concept only... an implementation is not required to implement it as described" -- the spec's own model for reasoning about the mechanism, not a mandated on-chip structure.) On receiving a new MSN, the requester walks its outstanding WQEs starting from the oldest and retires every one whose SSN is covered by the new value. Because a responder may coalesce acknowledgments, one incoming MSN can retire several WQEs in a single event -- the spec's own worked example has three ordinary SENDs, no READ or ATOMIC among them, and one ACK carrying MSN=3 completing all three at once.

**The wrinkle an outstanding READ or ATOMIC creates.** RDMA READ and ATOMIC both require an explicit response before the WQE that requested them can retire -- a plain SEND or WRITE doesn't need one, since its own eventual ACK is that response. §9.7.7.1.1's own worked example makes the resulting case concrete: a requester posts two ordinary SENDs, an RDMA READ, and a third SEND, in that order; the responder completes that third SEND (MSN advances to 4: four request messages -- the two earlier SENDs, the READ, which the responder counted when it validated the request under the early-increment option above, and this third SEND) *before* it finishes returning the READ's own response data, because RDMA READs are loosely ordered relative to other request types. Even so, the requester can only retire WQEs up to the first still-outstanding READ or ATOMIC in its list -- not past it, no matter what the MSN value alone would suggest about work completed behind it. The rule isn't "MSN covers my SSN, so I'm done" on its own; it's "MSN covers my SSN, *and* nothing ahead of me in post order is still waiting on its own explicit response." What the requester does about that READ is a separate question from retirement: an acknowledgment whose PSN lies past a still-outstanding READ or ATOMIC can only mean that request's response went missing, and Linux's software RoCE driver, for one, treats it as a retry trigger and resends from that READ rather than waiting for data that is not coming.

Play the whole case, from the four posts to the last retirement:

{{< flow "msn-retirement" >}}


> [!NOTE] Concept
> A completion never crosses the wire. The master diagram's TX lane and RX lane are both internal to one physical RNIC -- the peer at the other end of a connection is a separate, identically-shaped chip this diagram doesn't draw. Every transaction produces two completions, not one: this side's own send-side CQE, generated locally the moment this side's own reliability logic confirms MSN coverage for a retiring WQE, and the peer's own receive-side CQE, generated locally on the peer's own hardware the moment its own DMA writeback lands a posted Receive Request's data. The only thing that crosses between them is the acknowledgment itself -- never a completion record.

**Where this connects.** A WQE that clears both conditions above is done, in the sense [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor) already means by that word -- ready for [CQE Generation](/cq/cqe-generation) to act on. That chapter's own signal-flag gate (does this specific Work Request even want a completion reported) sits downstream of the decision this chapter makes, not upstream of it: this chapter decides *that* a Work Request is done; CQE Generation decides only whether *reporting* it is switched on.

> [!CAUTION] Gotcha
> Don't conflate this chapter with [The Replay Buffer](/reliability/the-replay-buffer)'s retirement logic just because both happen when an ACK arrives. Replay-buffer retirement is PSN-driven and frees a per-packet resend record; WQE completion is MSN-driven and closes out a per-message unit of application-visible work. They usually track closely together, but the READ/ATOMIC case above is a real, spec-documented instance where they diverge -- a later message can be MSN-complete while an earlier-posted WQE's packets are still legitimately outstanding.

> [!CAUTION] Gotcha
> IBTA-V1 defines a second, materially different MSN mechanism for Reliable Datagram service (§9.7.8, EE-context-addressed, tied to a RESYNC operation §9.4.2 states plainly is "supported only for the Reliable Datagram transport service"). RoCEv2 doesn't implement RD -- only RC. Reading the spec's RD sections directly for background is fine, but don't carry RESYNC or EE-context MSN recalculation into this chapter's model; RC's generation and retirement rules above are the complete, correct picture for this book's scope.

> [!TIP] Why
> A message-granularity counter is the cheaper design specifically because the responder already knows, at zero extra cost, exactly when it finished executing a given request -- it's the same event that already triggers building that request's own response. Handing the requester a small monotonic count of that event is far cheaper than the alternative: making the requester reconstruct message boundaries itself by counting how many packets each of its own WQEs needed, then matching that count against a stream of per-packet ACKs. It also buys the coalescing property above for free -- a responder that has fallen behind and is only now getting around to acknowledging can report several completed messages in a single packet, and the requester retires all of them from one arrival instead of needing one ACK per WQE.

## Interface reference

| Field | Width | Carried in | Generated | Consumed by |
|---|---|---|---|---|
| **MSN** | 24 bits, modulo 2^24 | AETH, alongside Syndrome | responder, once per completed non-duplicate request message | requester's WQE-retirement logic |
| **SSN** | logical concept only, no fixed wire or storage width | never wire-carried -- a per-QP requester-side bookkeeping concept | assigned to each WQE at post time | compared against incoming MSN to decide which WQEs retire |

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications), Release 1.2.1 -- §9.3.5.2 (MSN field definition), §9.7.7.1 and §9.7.7.1.1 (RC-scoped generation and requester retirement rules, including compliance statements C9-147 through C9-149). Read directly at these section numbers, not from a secondary source.
- This chapter's scope is Reliable Connection only, matching RoCEv2. IBTA-V1 §9.7.8's Reliable Datagram-specific MSN and RESYNC mechanics are a documented, real, but architecturally separate mechanism this book does not cover.
