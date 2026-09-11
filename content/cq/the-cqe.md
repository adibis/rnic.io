---
title: "The CQE"
weight: 1
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "A WQE is deliberately thin: hardware produces everything downstream of it. A CQE runs that discipline in reverse -- it's hardware's own report of what actually happened, and on a failed completion, most of it isn't even required to be trustworthy. This chapter is the field list, and the one status check that decides which fields you're allowed to read at all."
---

## Overview

A Completion Queue Entry (CQE) is the completed counterpart to a WQE: hardware's report to software that a Work Request finished, landing in a Completion Queue rather than the Send or Receive Queue that produced the request in the first place. [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor) already established that a WQE is deliberately thin, carrying nothing hardware alone produces -- no BTH, no PSN, no ICRC. A CQE is that same discipline run in the opposite direction: it's what hardware tells software once its own side of the work is actually done, and not one field more.

> [!NOTE] Concept
> A WQE describes a request before anything has happened. A CQE describes an outcome after everything has. They sit on opposite ends of the same operation, in different rings, written by different sides of the software/hardware boundary.

## Locator

![The master RNIC block diagram with CQ Storage highlighted and every other block dimmed -- this chapter covers what one ring entry actually contains, not the ring structure itself.](/images/cq/01-the-cqe-locator.svg)

Every field below lives inside one entry of the Completion Queue -- the structure a later chapter in this Part covers. This chapter is about what's in that entry, not the ring holding it.

## Block diagram

![Eight CQE fields split into two groups by a real, sourced rule: WR ID, completion status, local QP number, and vendor error syndrome are valid on every completion regardless of outcome; completion opcode, byte count, the immediate-data/invalidated-rkey union, and flags are valid only when the completion status is SUCCESS -- everything in the second group is unspecified on a failed completion.](/images/cq/01-the-cqe-block.svg)

## Deep walkthrough

**What the standard actually requires, versus what a software-facing shape makes concrete.** IBTA-V1 specifies a Work Completion's required content at a semantic level, tied to outcome rather than opcode: a successful completion must report what happened and how much data moved; a failed one must report that it failed and enough context to say why, at whatever granularity the implementation chooses to expose. It does not specify a C struct, a field order, or a byte layout -- the same standard/software-shape distinction [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor) already drew for Work Requests applies here without modification.

**The software-facing shape, field by field.** A generic completion descriptor, the kind any transport-agnostic software API hands back to a polling application, carries eight fields: the WR ID matching this completion back to the Work Request that produced it, a completion status naming success or a specific failure kind, the local QP number this completion belongs to, a vendor error syndrome, a completion opcode, a byte count, an immediate-data-or-invalidated-rkey union, and a flags field. Four of those eight -- WR ID, status, QP number, and the syndrome -- are the only ones a failed completion is required to leave trustworthy. The other four only mean something once status confirms the operation actually succeeded.

> [!CAUTION] Gotcha
> Reading a failed completion's byte count, opcode, or immediate-data field as if it were meaningful is a real, easy mistake. It isn't defined on failure. Check status first, always -- everything else about a CQE is downstream of that one field.

**Why this book is confident about the failure-side rule specifically.** This isn't a hedge invented to sound careful -- it's a directly documented software-interface guarantee: when a completion's status is anything other than success, only the WR ID, the status itself, the QP number, and the vendor error syndrome are specified as valid; every other field's contents on that same completion are left unspecified. [Single-QP Error Handling](/qp/single-qp-error-handling) already established the trigger side of this -- hardware flushes both of a faulted QP's queues, "generating a real CQE for each entry, even though none of them succeeded." This chapter's contribution is the other half: what those flush-generated CQEs are actually allowed to say, and what they aren't.

**The completion opcode has its own vocabulary -- it isn't just the Work Request's opcode echoed back.** [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor)'s opcode table covers what a *requester* posts: SEND, RDMA WRITE, RDMA READ, an atomic. A completion opcode has to cover more ground than that, because a receive-side completion doesn't have a Work Request behind it to echo at all -- it reports an RQ entry that got consumed, not a WQE that got issued. A plain SEND arriving at the responder, or a WRITE carrying immediate data, produces a completion opcode that names *receiving* something, a value that never appears anywhere in the requester-side opcode table, because nothing on the requester side ever needed to express it.

> [!TIP] Why
> Giving receive-side completions their own distinct opcode values, rather than overloading the requester-side opcode set to also mean "this arrived," lets software (or hardware) tell a send-side completion from a receive-side one with a single cheap check, without needing to enumerate and compare against every possible requester-side opcode first. A shared QP posts to both a Send Queue and a Receive Queue, and its completions land in the same Completion Queue interleaved -- a polling loop has to sort that out fast, on every single entry it reads, so the opcode encoding is worth designing for that specific access pattern rather than treating "which opcode" as a flat, arbitrary enum.

**The immediate/invalidated-rkey union, reappearing on the other side of the wire.** [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor) already named this same union as something a *requester's* WQE can carry -- an immediate value, or an rkey to invalidate. This is where that value actually surfaces: not back to the requester that sent it, but to the *responder*, in the completion generated when that SEND or WRITE arrives. A requester's `solicit` flag and a responder's completion aren't two unrelated mechanisms -- the flag is a request made at post time, and a completion carrying this same union field, on the far end of the wire, is what fulfills it.

**Where this is checked, and what happens next.** Nothing about reading a CQE is a free-for-all -- software has to check completion status before trusting any of the success-only fields, the same discipline this chapter has already named twice. Polling itself consumes a Completion Queue in strict producer/consumer order, the same shape [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor) already established for a Send Queue, covered in full by the next chapter in this Part rather than re-derived here. What decides whether a freshly-written CQE also wakes an application via an event, instead of waiting to be found by polling, is this Part's own event-arbitration chapter, later.

## Field reference

Every field a CQE carries, and the one condition that decides whether it's safe to read at all.

| Field | Carries | Valid when | Standard's role |
|---|---|---|---|
| **WR ID** | matches this completion back to the Work Request that produced it | always | architect's choice -- the same identifier [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor)'s WR ID field established, echoed back here |
| **completion status** | success, or which specific failure kind | always | IBTA-V1 requires the outcome be reportable; the exact status vocabulary is architect's choice |
| **local QP number** | which QP this completion belongs to | always | architect's choice |
| **vendor error syndrome** | implementation-specific detail, meaningful only on failure | always present; meaningful only on failure | architect's choice -- IBTA-V1 doesn't require a specific syndrome vocabulary |
| **completion opcode** | which operation completed -- its own vocabulary, not the WQE's opcode echoed back | success only | IBTA-V1 requires the semantic distinction be reportable; this exact vocabulary is architect's choice |
| **byte count** | bytes actually transferred | success only | architect's choice |
| **immediate data / invalidated rkey** | the same union field the requester's WQE carried | success only | IBTA-V1 -- mirrors the WQE-side union this chapter already cross-references |
| **flags** | which of the optional fields above this specific CQE actually carries | success only | architect's choice |

UD-only addressing fields (a source QP number, service-level and path routing detail for the arriving datagram) exist in real completion interfaces but sit outside this book's first-pass RC scope, the same boundary [QP Context](/qp/qp-context) already drew for UD-only QP fields.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
- The complete CQE field list, the completion-opcode vocabulary split between send-side and receive-side values, and the exact failure-side validity rule (only WR ID, status, QP number, and vendor syndrome are guaranteed meaningful on a failed completion) are sourced to the public `ibv_poll_cq`/`ibv_wc` manual documentation, not a direct quotation of IBTA-V1's own text -- the underlying semantic requirement (a Work Completion interface capable of reporting outcome, identity, and a bounded failure detail) is consistent with, but not verified word-for-word against, the standard's own Work Completion definition.
