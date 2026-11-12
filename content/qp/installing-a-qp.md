---
title: "Installing a QP"
weight: 4
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "A QP exists in hardware before it can send or receive anything at all. This chapter covers the two steps that get it there: allocating its table entry, and the one transition that turns an allocated-but-inert entry into something bound to a port with its access rules already fixed."
prev: /qp/qp-caching
next: /qp/qp-state-machine
metaDescription: "How an RDMA Queue Pair is installed in hardware: allocating its QP Table entry, and the RESET to INIT transition that binds it to a port."
seoTitle: "Installing an RDMA Queue Pair"
---

## Overview

A QP doesn't arrive in hardware fully formed. IBTA-V1 requires only that a QP come into existence as an allocated, addressable, and inert resource -- a table entry with a QP number, present but not yet configured with anything a packet could use -- before the one gated transition, RESET→INIT, gives it a port and a fixed remote-access permission set. Nothing in the standard says how many QPs one allocation request produces, or what the allocation path looks like in silicon; that's an architectural degree of freedom, not a wire-format or verbs-API fact, and this chapter opens with it before naming any specific software call. Installing a QP is the combination of that allocation and RESET→INIT. Everything past INIT -- naming a peer, arming the QP to send -- is [QP State Machine](/qp/qp-state-machine)'s subject; this chapter stops at the first gate.

## Locator

![The master RNIC block diagram with the QP Table highlighted and every other block, including QP Cache, dimmed -- this chapter writes a new entry into that table for the first time.](/images/qp/04-installing-a-qp-locator.svg)

[The QP Table](/qp/the-qp-table) covered the structure a QP's context lives in once it exists. This chapter covers how it gets there in the first place.

## Block diagram

![Installing a QP: an allocation request assigns a QP number into a RESET-state table entry, then a RESET->INIT state-transition request either fills in port/partition-key/access-flags together or is rejected outright, leaving the entry unchanged in RESET.](/images/qp/04-installing-a-qp-block.svg)

## Deep walkthrough

**Allocation: the entry exists before it means anything.** Creating a QP is a resource-allocation event against [The QP Table](/qp/the-qp-table)'s indexing scheme: some entity assigns a QP number and marks the corresponding entry present and addressable in the RESET state, carrying no port binding, no access permissions, no peer. That's the entire allocation-time requirement IBTA-V1 places on the architecture. How many QP numbers one allocation request actually produces is a question the standard leaves open, not one it answers -- an adapter could just as reasonably grant exactly one QP number per request as it could grant a batch of them in a single allocation call, amortizing whatever fixed per-request overhead the allocation path carries (a command descriptor, a doorbell ring, a firmware round trip) across many QPs a workload knows up front it's about to need, an MPI job or a distributed-training job establishing dozens of peer connections at startup being the obvious case. Nothing in the public record documents a shipping RNIC doing the latter, and this book doesn't assert one does; the point is that "one QP per allocation" is a plausible software-interface convention, not an architectural ceiling -- the dominant software model for requesting a QP asks for exactly one per call, but that's a fact about a software contract, not evidence that the hardware resource allocator underneath it is incapable of granting more than one QP at a time. A QP in RESET cannot legally send or receive anything; RESET exists specifically to hold a QP that's been allocated but not yet configured, and it's also the state a QP returns to when it's later torn down and rebuilt (per [QP State Machine](/qp/qp-state-machine)'s ERROR→RESET path).

**RESET→INIT: the first thing hardware actually enforces.** This is the QP's first state-transition request, and IBTA-V1 itself specifies exactly which attributes a Reliable Connection QP must supply at this transition -- the requirement is the standard's own state-machine definition. For RC, that's exactly four attributes: the state target itself, the partition-key index attribute, the port-binding attribute, and the remote-access permission mask. Each does specific, necessary work:

- **The port-binding attribute** places the QP on one of the adapter's physical ports -- which wire this QP's traffic will actually go out and come in on. A QP has to belong to exactly one port before it can be associated with any path to a peer.
- **The partition-key index attribute** selects which partition key this QP presents -- on native InfiniBand, where a subnet manager actively assigns and enforces partitions, this is a live access-control boundary checked well before any specific peer is named. This book is about RoCEv2 specifically, and RoCEv2 has no subnet manager: there's no fabric-wide authority assigning non-default partitions, so in practice almost every RoCEv2 deployment runs every QP against the default, full-membership P_Key. The field is still required here -- the wire format carries it, or the RC opcode catalog's BTH ([What Is RoCEv2](/overview/what-is-rocev2)) wouldn't have anywhere to put it -- but treat it as present for spec-compatibility with the field's IB origins, not as an active partitioning mechanism doing real access-control work in a typical RoCEv2 fabric.
- **The remote-access permission mask** fixes, up front, which remote operations this QP will honor against it once it's live: remote write, remote read, remote atomic, each independently enable-able. RESET→INIT is where this must first be supplied -- a QP is never allowed to exist in a state where its permission set is merely undefined -- but it isn't frozen there for the QP's whole life. Unlike the port-binding and partition-key attributes, it can be resupplied later: a same-state RTS→RTS transition that changes access flags on an already-live QP without touching the state target at all. A QP that wasn't granted remote-write access at INIT can be granted it later, live, in RTS; it's the port-binding and partition-key attributes -- not eligible for that same live update -- that can't be changed without tearing the QP down.

Supply all four and the QP moves to INIT. Supply fewer, and per [QP State Machine](/qp/qp-state-machine)'s general rule for every transition in this book, hardware refuses the call outright -- the entry stays in RESET, unchanged, not partially updated with whichever attributes did arrive.

> [!CAUTION] Gotcha
> There's no state where a QP is "half-installed" with a port but no access flags. Either the whole mask is satisfied, or none of it takes effect -- a rejected call never leaves partial attributes in place.

**What INIT still doesn't have.** A QP in INIT has a port and a permission set, but no peer -- no destination-QP field set, no path, no starting PSN. It cannot yet send or usefully receive; the next transition, INIT→RTR, is what actually connects it to somebody, and that transition, along with everything RTR→RTS adds on top, belongs entirely to [QP State Machine](/qp/qp-state-machine). This chapter's installation step ends here, deliberately: RESET→INIT is about making a QP a bound, permission-scoped object on the fabric; INIT→RTR and beyond are about connecting that object to a specific peer.

**Everything else that has to exist around it.** A QP doesn't get installed in isolation -- [One Transaction, Start to Finish](/overview/one-transaction-start-to-finish) already walked the fuller bring-up sequence this step sits inside: a Protection Domain, at least one registered Memory Region, and a Completion Queue typically all exist before or alongside QP creation, since a QP needs a PD to belong to and a CQ to report completions into before it's much use. None of that is re-derived here -- this chapter's scope is the QP's own installation step, not the objects around it.

> [!TIP] Why
> Requiring the remote-access permission mask at RESET→INIT rather than leaving it unset until some later, optional call closes off a real gap: without that requirement, a QP could reach a state where it's capable of exchanging packets while its permission set is simply undefined, which is exactly the kind of ambiguity a hardware-enforced gate exists to prevent. But unlike the port-binding and partition-key attributes -- which really do need a full teardown to change, since they have no equivalent live-update path at all -- access flags get a second, live path precisely because permission changes are a normal, expected operational need (granting remote-write access to a connection that didn't start with it, say) in a way that changing a QP's physical port or partition membership mid-connection isn't. The design didn't pick uniform rigidity across every RESET→INIT attribute; it picked rigidity for the two that define what a QP structurally *is*, and left a live door open for the one that's routinely operational.

That's every bit RESET→INIT requires, and nothing named above is optional -- omit one and the call is refused, per [QP State Machine](/qp/qp-state-machine), which picks up immediately where this chapter stops: INIT→RTR, naming a peer, and RTR→RTS, arming the QP to send.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
