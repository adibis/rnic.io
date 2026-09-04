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

<!-- IMAGE: Master RNIC block diagram, full canvas, with the "QP Table" block highlighted (full-opacity indigo override treatment per docs/SVG_STYLE.md, overriding its normal amber shared-resource color) and every other block, including QP Cache, at opacity 0.35. Caption: "Installing a QP writes a new entry into this table for the first time, then drives it through its first hardware-enforced transition." -->

[The QP Table](/qp/the-qp-table) covered the structure a QP's context lives in once it exists. This chapter covers how it gets there in the first place.

## Overview

A QP doesn't arrive in hardware fully formed. Software allocates a table entry for it, and that entry starts life in the RESET state -- present, addressable by its newly-assigned QP number, but not yet configured with anything a packet could use. Installing a QP is the combination of that allocation and the one transition, RESET→INIT, that turns an inert entry into one bound to a specific port with its remote-access rules already fixed. Everything past INIT -- naming a peer, arming the QP to send -- is [QP State Machine](/qp/qp-state-machine)'s subject; this chapter stops at the first gate.

## Block diagram

<!-- IMAGE: Left-to-right sequence of three boxes. Box 1: "ibv_create_qp() / equivalent hardware allocation call" with a small arrow labeled "assigns QP number" pointing down into Box 2. Box 2: "QP Table entry, state = RESET" (drawn in the amber shared-resource color, matching the-qp-table.md's block diagram) with fields shown as empty/greyed placeholders. An arrow labeled "ibv_modify_qp(RESET -> INIT), attribute mask: PKEY_INDEX | PORT | ACCESS_FLAGS" points right into Box 3: "QP Table entry, state = INIT" with those three fields now shown filled in (highlighted), the rest still greyed. A small red "X" branch off the transition arrow, labeled "mask incomplete -> call rejected, entry stays in RESET," per the fault/error role color. -->

## Deep walkthrough

**Allocation: the entry exists before it means anything.** Creating a QP is a resource-allocation request: software asks for one, and the adapter (through whatever driver interface fronts it, `ibv_create_qp` at the verbs layer) hands back a QP number and a table entry in the RESET state -- present and addressable, per [The QP Table](/qp/the-qp-table)'s indexing scheme, but carrying no port binding, no access permissions, no peer. A QP in this state cannot legally send or receive anything; RESET exists specifically to hold a QP that's been allocated but not yet configured, and it's also the state a QP returns to when it's later torn down and rebuilt (per [QP State Machine](/qp/qp-state-machine)'s ERROR→RESET path).

**RESET→INIT: the first thing hardware actually enforces.** This is the QP's first `ibv_modify_qp` call, and for Reliable Connection it requires exactly four attribute mask bits: `IBV_QP_STATE`, `IBV_QP_PKEY_INDEX`, `IBV_QP_PORT`, `IBV_QP_ACCESS_FLAGS`. Each does specific, necessary work:

- **`IBV_QP_PORT`** places the QP on one of the adapter's physical ports -- which wire this QP's traffic will actually go out and come in on. A QP has to belong to exactly one port before it can be associated with any path to a peer.
- **`IBV_QP_PKEY_INDEX`** selects which partition key this QP presents -- the partitioning mechanism that scopes which other QPs on the fabric it's even allowed to talk to, checked well before any specific peer is named.
- **`IBV_QP_ACCESS_FLAGS`** fixes, up front, which remote operations this QP will honor against it once it's live: remote write, remote read, remote atomic, each independently enable-able. This is a permission gate set once, here, not re-checked or re-negotiated later -- a QP that wasn't granted remote-write access at INIT cannot be made to accept one later without tearing it down and reinstalling it from RESET.

Supply all four and the QP moves to INIT. Supply fewer, and per [QP State Machine](/qp/qp-state-machine)'s general rule for every transition in this book, hardware refuses the call outright -- the entry stays in RESET, unchanged, not partially updated with whichever attributes did arrive. There's no state where a QP is "half-installed" with a port but no access flags; either the whole mask is satisfied, or none of it takes effect.

**What INIT still doesn't have.** A QP in INIT has a port and a permission set, but no peer -- no `dest_qp_num`, no path, no starting PSN. It cannot yet send or usefully receive; the next transition, INIT→RTR, is what actually connects it to somebody, and that transition, along with everything RTR→RTS adds on top, belongs entirely to [QP State Machine](/qp/qp-state-machine). This chapter's installation step ends here, deliberately: RESET→INIT is about making a QP a real, bound, permission-scoped object on the fabric; INIT→RTR and beyond are about connecting that object to a specific peer.

**Everything else that has to exist around it.** A QP doesn't get installed in isolation -- [One Transaction, Start to Finish](/overview/one-transaction-start-to-finish) already walked the fuller bring-up sequence this step sits inside: a Protection Domain, at least one registered Memory Region, and a Completion Queue typically all exist before or alongside QP creation, since a QP needs a PD to belong to and a CQ to report completions into before it's much use. None of that is re-derived here -- this chapter's scope is the QP's own installation step, not the objects around it.

> **Design rationale.** Fixing `qp_access_flags` at RESET→INIT rather than allowing it to be changed freely at any later point is a deliberate narrowing, not an oversight: remote-write and remote-atomic access are exactly the permissions a misbehaving or compromised peer would want to escalate mid-connection, and pinning them at the one transition hardware can still refuse outright closes that door before the QP is capable of exchanging a single packet. The cost is inflexibility that's visible to any application wanting to change a QP's permission profile without a connection interruption -- there is none available; the only path is tearing the QP down to RESET and reinstalling it with a new mask. That's the same trade [QP State Machine](/qp/qp-state-machine) makes at every other transition in this book's checklist: catch what you can at the gate, even at the cost of making a later change expensive.

## Interface reference

| Attribute (RC) | What it fixes | Changeable later without a full RESET? |
|---|---|---|
| `IBV_QP_PORT` | Physical port this QP is bound to | No |
| `IBV_QP_PKEY_INDEX` | Partition key scoping this QP's reachable peers | No |
| `IBV_QP_ACCESS_FLAGS` | Remote write / read / atomic permissions honored against this QP | No |
| `IBV_QP_STATE` | Confirms the transition itself (RESET→INIT) | -- |

That's every bit RESET→INIT requires, and nothing this table lists is optional -- omit one and the call is refused, per [QP State Machine](/qp/qp-state-machine), which picks up immediately where this chapter stops: INIT→RTR, naming a peer, and RTR→RTS, arming the QP to send.
