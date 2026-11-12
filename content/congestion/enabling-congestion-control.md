---
title: "Enabling Congestion Control"
weight: 4
date: 2026-09-12
publishDate: 2026-09-12
draft: false
description: "There's no congestion-control switch on a Queue Pair. There's a Traffic Class attribute -- the same QoS mechanism InfiniBand has always had -- and RoCEv2's own annex is what turns setting it correctly into the one act that actually makes a QP's packets eligible for the whole mechanism this Part describes."
prev: /congestion/caching
next: /congestion/cnp-generation
metaDescription: "There is no congestion-control switch on a Queue Pair. In RoCEv2 the Traffic Class attribute is what makes a QP's packets eligible for ECN and DCQCN."
seoTitle: "Enabling RoCEv2 Congestion Control on a QP"
---

## Overview

Nothing this book has covered so far installs a QP with an explicit "enable congestion control" flag, because no such flag exists. What actually gates whether a QP participates in this Part's mechanism at all is Traffic Class -- a native InfiniBand QP attribute this book hasn't needed to name until now -- and RoCEv2's own annex is what gives that attribute a second job: determining the DSCP value stamped on this QP's outgoing IP header, which is the one thing a switch actually needs to be willing to ECN-mark this QP's packets in the first place.

> [!NOTE] Concept
> Congestion control isn't switched on. It's a consequence of a QP attribute this book would have needed to name eventually regardless -- Traffic Class governs QoS classification whether or not congestion control exists at all, and RoCEv2's DSCP mapping is what makes that same attribute double as this Part's own on-ramp.

## Locator

![The master RNIC block diagram with Rate Limiter highlighted and every other block dimmed -- this chapter covers the handoff that gives a QP's rate-limiter context its starting values, not the ongoing mechanism that updates them.](/images/congestion/04-enabling-congestion-control-locator.svg)

[Caching](/congestion/caching) covered whether this resource needs a working set at all. This chapter is about how one QP's entry -- however it's ultimately stored -- gets its starting values in the first place.

## Block diagram

![A QP's Traffic Class attribute, carried in the address vector QP State Machine already documents INIT->RTR requiring alongside the destination-QP field and path MTU, mapped by RoCEv2's own Annex A17 onto the IP header's DSCP field -- the value a switch actually checks before ECN-marking a packet. In parallel, the rate-limiter entry itself starts at fixed, sourced values: current rate at full line rate if no other flow is active from this host, no slow start, and the reaction factor initialized to 1.](/images/congestion/04-enabling-congestion-control-block.svg)

## Deep walkthrough

**Traffic Class: not a new mechanism, a QP attribute this book would have needed eventually anyway.** InfiniBand has always had a Traffic Class attribute for QoS classification -- it predates RoCEv2 and has nothing to do with congestion control on its own. What RoCEv2's Annex A17 adds is a mapping: a QP's Traffic Class becomes the DSCP value carried in that QP's outgoing IP header. A switch deciding whether to ECN-mark a packet under load checks that DSCP value against its own configured thresholds -- the mechanism [CNP Generation](/congestion/cnp-generation) covers from the marked-packet-arrives side. A QP whose Traffic Class never gets set to a DSCP value any switch along its path treats as congestion-control-eligible traffic can still send and receive normally; it just never accumulates the ECN marks that would ever trigger a CNP back to it. There is no separate flag to check -- Traffic Class alone decides.

> [!TIP] Why
> Reusing an existing QoS attribute instead of inventing a dedicated toggle keeps the attribute sets [QP State Machine](/qp/qp-state-machine) already covers unchanged -- Traffic Class doesn't need a new gate of its own because it's arguably already there for QoS purposes independent of congestion control. RoCEv2 gets a working congestion-control on-ramp for free, from an attribute InfiniBand QoS already required software to think about.

> [!CAUTION] Gotcha
> Setting a sensible Traffic Class is necessary but not sufficient on its own -- the switches along this QP's actual path also have to be configured to recognize that DSCP value and apply ECN marking to it under load. This chapter's subject is entirely end-host; whether the fabric in between actually cooperates is a network-operator configuration question this book doesn't reach.

**Where this connects to QP State Machine -- and specifically not to Installing a QP's own transition.** [Installing a QP](/qp/installing-a-qp) covers RESET→INIT, which [QP State Machine](/qp/qp-state-machine) is explicit places a QP on a port with nothing about a peer settled yet. Traffic Class has no sensible home there -- it's carried in the address vector, the same path-to-the-peer attribute set [QP State Machine](/qp/qp-state-machine) already documents INIT→RTR requiring, alongside the destination-QP field and path MTU. That's the transition that actually names a peer and the route to it, which is exactly what a DSCP-relevant, path-specific attribute needs to exist before it means anything. This chapter doesn't add a new state transition or a new gate -- only a new attribute worth naming, at a transition this book had already covered, once congestion control is the subject.

**What a freshly-installed QP's rate-limiter entry actually starts as -- sourced, not assumed, and narrower than it first looks.** Two facts here come directly from the DCQCN paper rather than this book's own inference. First, the reaction factor's initial value is 1, stated directly as a footnote to the paper's own rate-reduction equation. Second, and more surprising to a reader used to TCP's own slow start: there is no slow-start phase at all, for the specific case the paper actually describes. A flow starts sending at full line rate the instant it has traffic to send, "if there are no other active flows from the host" -- but the paper's own footnote to that exact sentence narrows it immediately: "Otherwise, starting rate is defined by local QoS policies." Full line rate is the no-other-active-flows case specifically, not a blanket default for every new QP regardless of what else that host is already sending.

> [!WARNING] Analogy
> This is the opposite default from what a TCP-literate reader expects, in the case where it actually applies. TCP starts a connection cautiously and ramps up, because it has no other way to learn the path's capacity. A rate limiter here starts at full speed and only ever backs off in response to an actual, specific signal, when it's the only thing this host is sending -- optimizing for the common case (an uncongested fabric, which a well-provisioned datacenter fabric is supposed to be most of the time) rather than assuming congestion until proven otherwise. Once a host already has other active flows, the same paper hands the starting-rate decision to local QoS policy instead, not to this default.

> [!CAUTION] Gotcha
> No slow start is not this Part's only line of defense against loss during the window before a QP's first CNP can possibly land. The same paper is direct about this: DCQCN "does not obviate the need for PFC. With DCQCN, flows start at line rate. Without PFC, this can lead to packet loss and poor performance." PFC is out of this book's RNIC-architecture scope, but it's worth naming here specifically because this chapter's own design choice depends on it existing -- full-line-rate start is only safe because link-level PFC is still there to prevent the packet loss it would otherwise cause.

> [!IMPORTANT] Food for Thought
> A single QP starting alone gets full line rate; several QPs from the same host starting together fall under local QoS policy instead, per the paper's own footnote. If you were configuring that policy for a distributed training job whose collective operation starts dozens of QPs from one host at the same instant, would you still let them all start near full rate, betting on PFC to absorb the difference, or would you deliberately stagger them -- and what would that decision cost in the job's own startup latency?

**Where this connects.** Traffic Class is set once, at INIT→RTR, the same transition that names a peer at all. What actually changes those initial rate-limiter values after that point -- a Congestion Notification Packet arriving -- is [CNP Generation](/congestion/cnp-generation) and [The Rate Control State Machine](/congestion/the-rate-control-state-machine), both still ahead in this Part.

## References

- InfiniBand Trade Association, *Supplement to InfiniBand Architecture Specification Volume 1, Release 1.2.1, Annex A17: RoCEv2* (2014) -- governs the Traffic Class-to-DSCP mapping this chapter's central claim rests on.
- Zhu, Eran, Firestone, Guo, Lipshteyn, Liron, Padhye, Raindel, Haj Yahia, Zhang, "Congestion Control for Large-Scale RDMA Deployments," SIGCOMM 2015 -- the initial alpha value (footnote to their Equation 1), the no-slow-start design decision and its own footnoted local-QoS-policy exception, and the explicit PFC-still-required statement (§3.2), all quoted directly.
