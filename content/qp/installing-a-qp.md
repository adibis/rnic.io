---
title: "Installing a QP"
weight: 4
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "A QP exists in hardware before it can send or receive anything at all. This chapter covers the two steps that get it there: allocating its table entry, and the one transition that turns an allocated-but-inert entry into something bound to a port with its access rules already fixed."
prev: /qp/qp-caching
next: /qp/qp-state-machine
---

## Locator

![The master RNIC block diagram with the QP Table highlighted and every other block, including QP Cache, dimmed -- this chapter writes a new entry into that table for the first time.](/images/qp/04-installing-a-qp-locator.svg)

[The QP Table](/qp/the-qp-table) covered the structure a QP's context lives in once it exists. This chapter covers how it gets there in the first place.

## Overview

A QP doesn't arrive in hardware fully formed. Software allocates a table entry for it, and that entry starts life in the RESET state -- present, addressable by its newly-assigned QP number, but not yet configured with anything a packet could use. Installing a QP is the combination of that allocation and the one transition, RESET→INIT, that turns an inert entry into one bound to a specific port with its remote-access rules already fixed. Everything past INIT -- naming a peer, arming the QP to send -- is [QP State Machine](/qp/qp-state-machine)'s subject; this chapter stops at the first gate.

## Block diagram

<!-- IMAGE: Left-to-right sequence of three boxes. Box 1: "ibv_create_qp() / equivalent hardware allocation call" with a small arrow labeled "assigns QP number" pointing down into Box 2. Box 2: "QP Table entry, state = RESET" (drawn in the amber shared-resource color, matching the-qp-table.md's block diagram) with fields shown as empty/greyed placeholders. An arrow labeled "ibv_modify_qp(RESET -> INIT), attribute mask: PKEY_INDEX | PORT | ACCESS_FLAGS" points right into Box 3: "QP Table entry, state = INIT" with those three fields now shown filled in (highlighted), the rest still greyed. A small red "X" branch off the transition arrow, labeled "mask incomplete -> call rejected, entry stays in RESET," per the fault/error role color. -->

## Deep walkthrough

**Allocation: the entry exists before it means anything.** Creating a QP is a resource-allocation request: software asks for one, and the adapter (through whatever driver interface fronts it, `ibv_create_qp` at the verbs layer) hands back a QP number and a table entry in the RESET state -- present and addressable, per [The QP Table](/qp/the-qp-table)'s indexing scheme, but carrying no port binding, no access permissions, no peer. A QP in this state cannot legally send or receive anything; RESET exists specifically to hold a QP that's been allocated but not yet configured, and it's also the state a QP returns to when it's later torn down and rebuilt (per [QP State Machine](/qp/qp-state-machine)'s ERROR→RESET path).

**RESET→INIT: the first thing hardware actually enforces.** This is the QP's first `ibv_modify_qp` call, and for Reliable Connection it requires exactly four attribute mask bits: `IBV_QP_STATE`, `IBV_QP_PKEY_INDEX`, `IBV_QP_PORT`, `IBV_QP_ACCESS_FLAGS`. Each does specific, necessary work:

- **`IBV_QP_PORT`** places the QP on one of the adapter's physical ports -- which wire this QP's traffic will actually go out and come in on. A QP has to belong to exactly one port before it can be associated with any path to a peer.
- **`IBV_QP_PKEY_INDEX`** selects which partition key this QP presents -- on native InfiniBand, where a subnet manager actively assigns and enforces partitions, this is a live access-control boundary checked well before any specific peer is named. This book is about RoCEv2 specifically, and RoCEv2 has no subnet manager: there's no fabric-wide authority assigning non-default partitions, so in practice almost every RoCEv2 deployment runs every QP against the default, full-membership P_Key. The field is still required here -- the verbs API and the wire format both carry it, or the RC opcode catalog's BTH ([0.2](/overview/what-is-rocev2)) wouldn't have anywhere to put it -- but treat it as present for spec-compatibility with the field's IB origins, not as an active partitioning mechanism doing real access-control work in a typical RoCEv2 fabric.
- **`IBV_QP_ACCESS_FLAGS`** fixes, up front, which remote operations this QP will honor against it once it's live: remote write, remote read, remote atomic, each independently enable-able. RESET→INIT is where this must first be supplied -- a QP is never allowed to exist in a state where its permission set is merely undefined -- but it isn't frozen there for the QP's whole life: the kernel's own RC state-transition table (`ib_modify_qp_is_ok`'s `qp_state_table`, in `drivers/infiniband/core/verbs.c`) lists `IB_QP_ACCESS_FLAGS` as a legal optional attribute on an RTS→RTS call -- a same-state `ibv_modify_qp` that changes access flags on an already-live QP without touching `qp_state` at all. A QP that wasn't granted remote-write access at INIT can be granted it later, live, in RTS; it's `IBV_QP_PORT` and `IBV_QP_PKEY_INDEX` -- not carried in that same RTS→RTS optional set -- that genuinely can't be changed without tearing the QP down.

Supply all four and the QP moves to INIT. Supply fewer, and per [QP State Machine](/qp/qp-state-machine)'s general rule for every transition in this book, hardware refuses the call outright -- the entry stays in RESET, unchanged, not partially updated with whichever attributes did arrive. There's no state where a QP is "half-installed" with a port but no access flags; either the whole mask is satisfied, or none of it takes effect.

**What INIT still doesn't have.** A QP in INIT has a port and a permission set, but no peer -- no `dest_qp_num`, no path, no starting PSN. It cannot yet send or usefully receive; the next transition, INIT→RTR, is what actually connects it to somebody, and that transition, along with everything RTR→RTS adds on top, belongs entirely to [QP State Machine](/qp/qp-state-machine). This chapter's installation step ends here, deliberately: RESET→INIT is about making a QP a real, bound, permission-scoped object on the fabric; INIT→RTR and beyond are about connecting that object to a specific peer.

**Everything else that has to exist around it.** A QP doesn't get installed in isolation -- [One Transaction, Start to Finish](/overview/one-transaction-start-to-finish) already walked the fuller bring-up sequence this step sits inside: a Protection Domain, at least one registered Memory Region, and a Completion Queue typically all exist before or alongside QP creation, since a QP needs a PD to belong to and a CQ to report completions into before it's much use. None of that is re-derived here -- this chapter's scope is the QP's own installation step, not the objects around it.

> **Design rationale.** Requiring `qp_access_flags` at RESET→INIT rather than leaving it unset until some later, optional call closes off a real gap: without that requirement, a QP could reach a state where it's capable of exchanging packets while its permission set is simply undefined, which is exactly the kind of ambiguity a hardware-enforced gate exists to prevent. But unlike `IBV_QP_PORT` and `IBV_QP_PKEY_INDEX` -- which really do need a full teardown to change, since they're not on the RTS→RTS optional list at all -- access flags get a second, live path precisely because permission changes are a normal, expected operational need (granting remote-write access to a connection that didn't start with it, say) in a way that changing a QP's physical port or partition membership mid-connection isn't. The design didn't pick uniform rigidity across every RESET→INIT attribute; it picked rigidity for the two that define what a QP structurally *is*, and left a live door open for the one that's routinely operational.

## Interface reference

| Attribute (RC) | What it fixes | Changeable later without a full RESET? |
|---|---|---|
| `IBV_QP_PORT` | Physical port this QP is bound to | No |
| `IBV_QP_PKEY_INDEX` | Partition key index -- an active partitioning boundary on native IB; carried for spec-compatibility in most RoCEv2 fabrics, which run no subnet manager to assign non-default partitions | No |
| `IBV_QP_ACCESS_FLAGS` | Remote write / read / atomic permissions honored against this QP | Yes -- via a same-state RTS→RTS `ibv_modify_qp` call |
| `IBV_QP_STATE` | Confirms the transition itself (RESET→INIT) | -- |

That's every bit RESET→INIT requires, and nothing this table lists is optional -- omit one and the call is refused, per [QP State Machine](/qp/qp-state-machine), which picks up immediately where this chapter stops: INIT→RTR, naming a peer, and RTR→RTS, arming the QP to send.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
