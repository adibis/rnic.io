---
title: "The Completion Queue"
weight: 2
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "A Send Queue is software producing, hardware consuming. A Completion Queue is the exact same ring shape with both roles swapped -- hardware produces, software consumes -- and it's usually not even feeding from one QP: many QPs' completions land in the same ring, interleaved in whatever order they actually finish in."
prev: /cq/the-cqe
next: /cq/creating-a-completion-queue
metaDescription: "An RDMA Completion Queue is a ring like the Send Queue with the roles swapped: hardware writes CQEs, software reads them, often for many QPs at once."
seoTitle: "The RDMA Completion Queue Ring"
---

## Overview

[The CQE](/cq/the-cqe) covered what one entry contains. This chapter covers the ring holding them: a fixed-depth structure in host memory, created once, that many QPs can point at simultaneously -- unlike a Send Queue, which belongs to exactly one QP for its entire life.

## Locator

![The master RNIC block diagram with CQ Storage highlighted and every other block dimmed -- this chapter covers the ring itself, not the entries inside it.](/images/cq/02-the-completion-queue-locator.svg)

Every CQE from the previous chapter lives inside one of these rings. This chapter is about the ring's own shape -- how it's addressed, who writes to it, who reads from it, and what happens when several QPs share one.

## Block diagram

![A single Completion Queue holding five written CQEs from three different QPs (A, B, A, C, B) interleaved in strict arrival order, three empty slots ahead of them, a consumer index that software's polling advances and a producer index that CQE Generation advances -- below it, a direct side-by-side contrast with the Send Queue Ring showing the producer/consumer roles fully reversed.](/images/cq/02-the-completion-queue-block.svg)

## Deep walkthrough

**The same ring shape as a Send Queue, with both roles reversed.** [The Send Queue Ring](/tx/the-send-queue-ring) is a structure software produces into and hardware consumes from -- software posts a WQE, hardware fetches it. A Completion Queue is the identical ring shape with the roles fully swapped: hardware produces a CQE the instant an operation resolves, and software consumes it by polling. Nothing about the underlying mechanism changed between the two; what changed is which side of the software/hardware boundary is allowed to write and which side is only allowed to read.

> [!TIP] Why
> The reversal isn't arbitrary -- it follows directly from which side of the boundary actually knows the thing being recorded. A Send Queue records intent, and only software knows what work it wants done, so software has to be the producer. A Completion Queue records outcome, and only hardware knows what actually happened to an operation once it's left host memory -- whether it landed, timed out, or hit a permission violation on the wire -- so hardware has to be the producer there instead. Same ring, same producer/consumer discipline, opposite assignment of who's allowed to be which, because the information itself originates on opposite sides.

**Addressed by ring position, not by QP.** [The QP Table](/qp/the-qp-table) is addressed by QP number, and [The Replay Buffer](/reliability/the-replay-buffer) by QP and PSN together -- both address by *which QP* a piece of state belongs to. A Completion Queue doesn't: it's addressed as a CQ handle plus a position in that CQ's own ring, full stop. Which QP a given entry actually reports on is data carried *inside* the CQE itself ([The CQE](/cq/the-cqe)'s own local-QP-number field), not part of how the ring is indexed. That's what makes sharing one CQ across many QPs a simple, structural fact rather than a special case needing its own addressing scheme: the ring doesn't care whose completion it's holding.

**Many QPs, one ring, interleaved in arrival order -- confirmed, not assumed.** A Completion Queue is a strict FIFO: entries come out in the order they went in. What it does *not* guarantee is anything about the relative order completions from different QPs arrive in when several QPs share one CQ -- documented directly rather than inferred: when multiple queues feed one CQ, "the order of generation for the completion queue entries... cannot be guaranteed" across those queues, only within the ring's own strict insertion order once an entry actually lands. [Multi-QP Error Arbitration](/qp/multi-qp-error-arbitration) already assumed this exact scenario as fact -- "a CQ or SRQ shared across dozens or hundreds of QPs" -- and this is the structural reason that sharing is unremarkable: the ring doesn't distinguish QPs at all, so adding another one costs nothing beyond that QP's own share of the fixed depth.

**Depth is fixed at creation, and the number software asked for isn't necessarily the number it got.** A Completion Queue's ring depth is set once, at creation, and doesn't grow or shrink on its own afterward. Documented directly: a real creation call takes a requested minimum depth and is explicitly permitted to allocate *more* than that -- rounding up to whatever boundary the implementation finds convenient -- which means software has to read back the actual allocated depth rather than assume the number it asked for is the number it received. *Creating a Completion Queue*, later in this Part, covers that negotiation in full; this chapter's job is just naming that the ring's capacity is a fixed, queryable fact once creation returns, not a moving target.

**What happens when the ring is full.** A CQE-writing event can arrive with no empty slot left to write into -- the producer index has caught all the way up to the consumer index, and software hasn't polled fast enough to make room. This chapter doesn't answer what happens next; *CQ Overflow*, later in this Part, is where that failure mode gets its own full treatment.

**What this ring is not.** It's not the mechanism that decides *when* a CQE gets written -- that's *CQE Generation*'s subject, fed by both the TX and RX pipelines converging on this one structure, and a chapter still ahead in this Part. And it's not the entry format itself, already covered. This chapter's scope stops at the ring's own shape: fixed depth, reversed producer/consumer roles, addressed by position rather than by QP.

## References

- No IETF or IBTA standard governs a Completion Queue's internal ring layout or addressing scheme -- it is implementation-specific silicon and software-interface design, not a wire-format contract any standards body defines. This chapter's depth-negotiation and cross-QP ordering claims are grounded in public completion-queue creation and polling documentation, cited in-text above, not a standards document.
