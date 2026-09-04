---
title: "Permission and Bounds Violations"
weight: 5
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "An RDMA write that lands outside a memory region's registered range and one that lands inside it but requests an operation the region was never registered to allow surface through the exact same completion status -- no public source says these are two separate hardware checks, or which one runs first."
prev: /mr/registering-a-memory-region
next: /mr/shared-mr-arbitration
---

## Locator

![The master RNIC block diagram with MR/Key Table and Key Cache highlighted -- this chapter is about a lookup at this pair that comes back rejected, not one that misses.](/images/mr/05-permission-and-bounds-violations-locator.svg)

[Registering a Memory Region](../registering-a-memory-region) covered how an MR gets its base address, length, and access-permission bits installed into the key table in the first place. This chapter is about what happens when an already-registered entry gets looked up by an incoming remote operation that doesn't fit it — either the address and length it names fall outside what was registered, or the operation type it's asking for isn't one the registration allowed.

## Overview

Two distinct real-world conditions are this chapter's subject: a bounds violation, where the address-plus-length an incoming RDMA operation names falls outside the MR's registered range, and a permission violation, where the operation type itself — a remote write, say — isn't enabled by the flags the MR was registered with (an MR registered with only `IBV_ACCESS_REMOTE_READ` has no obligation to accept a remote write). Both are named conditions, both are real, and both are worth stating precisely rather than folding into one vague "bad remote access" idea. What isn't real, or at least isn't documented anywhere public, is a hardware distinction between them: every source available treats a rejected remote access as one umbrella status, regardless of which of the two conditions actually triggered it. This chapter says that plainly rather than inventing a check order the specification doesn't commit to.

## Block diagram

<!-- IMAGE: A single incoming RDMA request box (labeled "incoming remote op: rkey, address, length, opcode") on the left, with an arrow into a "Key Table / MPT entry lookup" box in the center. From that box, two parallel branches drawn side by side (not sequential, no arrow from one to the other) labeled "bounds check: address+length within registered range?" and "permission check: opcode enabled by access flags?", joined by a dashed bracket underneath both labeled "check order not documented -- drawn side by side deliberately, not as a pipeline." Both branches converge into a single "pass/fail" diamond. The fail path forks into two boxes on the right: one on the responder's own QP labeled "IBV_EVENT_QP_ACCESS_ERR (this QP -> ERROR)" and one reached via a NAK arrow crossing a dotted "wire" boundary into the requester's QP, labeled "IBV_WC_REM_ACCESS_ERR (requester's completion)." -->

## Deep walkthrough

**The two conditions, named precisely.** A bounds violation is an addressing problem: the incoming operation's virtual address and length, taken together, reach past either end of the range the MR registered — reading or writing bytes the application never handed to `ibv_reg_mr` in the first place. A permission violation is an authorization problem on an address range that's otherwise entirely valid: the bytes are inside the MR, but the requested operation isn't one the registration's access flags enabled — a remote write against `IBV_ACCESS_REMOTE_READ`-only memory, a remote atomic against an MR that never set `IBV_ACCESS_REMOTE_ATOMIC`. These are conceptually independent failures. An operation can violate bounds without touching permissions at all (a well-permissioned MR, wrong offset), and it can violate permissions on a perfectly in-range address (the right bytes, the wrong verb).

**One umbrella status covers both, and that's not this book inventing a simplification.** `IBV_EVENT_QP_ACCESS_ERR` — the responder-side async event Part 0's research already documents — lists its causes as an R_Key violation, a misaligned atomic request, too many outstanding RDMA read/atomic requests, and a length error on an operation without immediate data. Nothing in that list, or anywhere else checked, separates "the address was out of range" from "the operation type wasn't allowed" into two distinct causes with two distinct codes. `IBV_WC_REM_ACCESS_ERR` — the completion status the *requester's* QP sees, from Part 1's research into the `ibv_wc_status` enum — is the same story from the other side of the wire: one status, covering the same class of remote-side rejection, regardless of which specific condition the responder's hardware actually tripped on.

**This is a genuinely two-sided event, not a one-QP fault.** [Single-QP Error Handling](/qp/single-qp-error-handling) is about one QP's own local fault and its own local flush. A permission or bounds violation isn't that shape: the check happens on the *responder's* side, against the *responder's* MR, and it's the responder's own QP that fires `IBV_EVENT_QP_ACCESS_ERR` and auto-transitions to ERROR — at which point everything that chapter already covers about flushing outstanding work with `IBV_WC_WR_FLUSH_ERR` applies to the responder's QP exactly as written. But the *requester* — a different QP, possibly on a different machine entirely — never sees that event. What it sees is a NAK crossing the wire and a completion carrying `IBV_WC_REM_ACCESS_ERR` on its own send queue. Two QPs, two local artifacts, one underlying violation. Worth stating precisely rather than letting the single-QP chapter's framing quietly stand in for this one.

**No documented check order, and no invented one here.** Does hardware check bounds first and only evaluate permissions on an in-range address, or the reverse? No source found states an order, and none states that the two are even implemented as separate stages rather than one combined table-entry validation that either passes or doesn't. Given that the only software-visible signal either way is the same umbrella status, there's no observable behavior that would let an application (or this book) infer an order empirically. The honest position: this is an implementation detail the spec doesn't mandate a visible order for, and this chapter isn't going to assert one anyway.

**Stale keys are a software problem, not a hardware-guaranteed-safe one.** A related but distinct failure mode: a remote peer holding an rkey to an MR that's since been deregistered. RDMAmojo's `ibv_dereg_mr()` documentation is explicit that this is the application's problem to avoid, not hardware's problem to catch: "It is up to the user to stop using the keys (lkey and rkey) that are associated with this MR. If those keys will be used in local Work Requests or by remote operation requests, it will lead to Work Completion errors." That's a software contract, stated plainly — synchronize before deregistering — not a claim that hardware detects every possible stale-key race and rejects it cleanly. What a stale-key access actually produces at the hardware level, in a badly-sequenced race where deregistration and a remote access to the same rkey happen close together, isn't stated anywhere found. [Mid-Transaction Deregistration](../mid-transaction-deregistration) picks this exact race up in depth; this chapter's job is just naming it as real and distinct from the bounds/permission checks above, not re-deriving it.

> **Design rationale.** Collapsing bounds and permission checks into one reported status, rather than exposing two, costs an application almost nothing in practice — either way, the right response to a remote access error is the same: stop, figure out (in software, from context the RNIC never had reason to hand back) what was actually wrong with the request, and fix the program that issued it. A finer-grained status would help a developer debugging one specific mistake, but it would also mean carrying two codes down every path that currently only needs one, for a distinction whose only consumer is a human reading a log, not a machine making a different decision based on which one fired. That's a plausible reason the umbrella status exists as one thing rather than two — reasoned here, not confirmed by any source, since no source explains the design decision either way.

## Interface and timing reference table

| Signal / status | Fires when | Scope |
|---|---|---|
| `IBV_EVENT_QP_ACCESS_ERR` | R_Key violation, misaligned atomic request, too many outstanding RDMA read/atomic requests, or a length error without immediate data | responder-side QP, RC only; auto-transitions that QP to ERROR |
| `IBV_WC_REM_ACCESS_ERR` | requester's completion status when the remote side rejected the operation for any of the above reasons | requester-side QP, via its own completion queue |
| Bounds violation (address+length outside registered range) | not a separately reported condition | folds into the umbrella statuses above |
| Permission violation (opcode not enabled by access flags) | not a separately reported condition | folds into the umbrella statuses above |
| Check ordering (bounds-then-permission vs. the reverse) | not documented by any public source | implementation detail, not spec-mandated |
| Stale rkey after deregistration | application's responsibility per `ibv_dereg_mr()` contract; not hardware-guaranteed-safe | see [Mid-Transaction Deregistration](../mid-transaction-deregistration) |

The two conditions this chapter named are what happens when one QP's access to one MR is rejected. [Shared-MR Arbitration](../shared-mr-arbitration) asks the harder question underneath that: what happens when many QPs are addressing that same MR at once, and one of them is trying to invalidate it while the others are still mid-lookup.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
