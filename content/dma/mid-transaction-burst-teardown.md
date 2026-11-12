---
title: "Mid-Transaction Burst Teardown"
weight: 9
date: 2026-09-16
publishDate: 2026-09-16
draft: false
description: "Two already-published chapters named this exact resource and left the same honest question open: what happens when the ground a running transfer depends on -- the QP that owns it, or the Memory Region backing its translation -- gets pulled out from under it mid-execution. This chapter doesn't resolve either question more cleanly than they already declined to. It states them together, and says why."
prev: /dma/multi-qp-burst-arbitration
metaDescription: "What happens to a running DMA transfer when its QP is reset or its Memory Region is deregistered mid-burst, and why no public source settles it."
seoTitle: "Tearing Down a DMA Burst Mid-Transaction"
---

## Overview

A transfer this engine is actively running depends on two things staying put for its whole duration: the Queue Pair that owns it, and the translation backing whatever address it's moving bytes through. Two already-published chapters each flagged one half of what happens when that assumption breaks, and both declined to resolve it. This chapter states both halves together, in this engine's own terms, rather than inventing a cleaner answer either chapter already chose not to give.

## Locator

![The master RNIC block diagram with DMA Engine highlighted and every other block dimmed -- this chapter is this Part's own version of the recurring mid-transaction-reset question every prior Part in this book has had to answer for its own resource.](/images/dma/09-mid-transaction-burst-teardown-locator.svg)

## Block diagram

![One in-flight transfer with two independent dashed lines striking into it, each ending in a question mark: one from a QP RESET event, asking whether the transfer is allowed to drain to completion or gets aborted immediately; one from a concurrent MR deregistration event, asking whether the transfer's own translation was a private copy immune to the invalidation or a live reference to a table entry that just changed underneath it.](/images/dma/09-mid-transaction-burst-teardown-block.svg)

## Deep walkthrough

**A transfer that hasn't started yet is the easy case.** A request still waiting in [Multi-QP Burst Arbitration](/dma/multi-qp-burst-arbitration)'s queue, with no slot yet and no bytes yet moved, can simply be discarded the moment its owning QP resets or the MR it targets gets deregistered -- nothing has executed, so nothing is torn. Everything below is about the harder case: a transfer already running, mid-segment, when either event lands.

**Trigger one: the owning QP resets mid-burst.** [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) already named this directly -- "a DMA write that may be actively mid-burst into host memory this instant, not yet committed in full" is one of four satellite structures a QP RESET has to reconcile, without describing the reconciliation. Two shapes of answer are available, the same honest framing [Mid-Transaction Deregistration](/mr/mid-transaction-deregistration) already used for its own open question. One: RESET waits, letting any transfer already attributed to this QP drain to completion or to a reported fault before the QP is considered fully reset -- safe, at the cost of a RESET whose own latency now scales with however long the longest in-flight transfer takes to finish. Two: RESET aborts in-flight transfers immediately, at whatever point they've reached -- fast, at the cost of a torn write, the same no-rollback outcome [Single-Burst Fault Handling](/dma/single-burst-fault-handling) already established this engine has no undo mechanism for. This book cannot resolve which shape any real implementation chose, and says so plainly rather than picking the more comforting answer.

**Trigger two: the backing Memory Region gets deregistered mid-burst.** [Mid-Transaction Deregistration](/mr/mid-transaction-deregistration) already covers this in full, from the Key Table's own side: does this engine hold a private, per-burst translation copy immune to a concurrent invalidation, or does it re-touch the shared table or cache at some granularity during a long transfer and risk a genuine race? [Address Translation Handoff](/dma/address-translation-handoff) already forward-referenced this exact question from this engine's own vantage point, and this chapter doesn't reopen it -- "this book cannot resolve which of the two is true for any real RNIC" stands exactly as that chapter already stated it.

> [!CAUTION] Gotcha
> Don't treat these two triggers as the same question asked twice. A QP reset is a *local* control-plane event, initiated deliberately by software managing that one connection. An MR deregistration is a *different* control-plane event, on a different object, that can be triggered by software that has no idea a transfer using that MR is even running. They can also compound: a QP reset and an MR deregistration racing against the same in-flight transfer simultaneously is a real possibility neither already-published chapter's treatment, nor this one, extends to.

**Why this book states both this way rather than guessing.** Every other Part's mid-transaction chapter has hit the same wall: no public source documents how a real RNIC actually behaves at these exact seams, because the seams themselves are invisible from outside the chip -- software sees a completed reset or a completed deregistration call, not the microarchitectural sequencing that got it there safely (or didn't). This book's position, consistent across every Part that has reached this question, is to name the real hazard precisely and say plainly what isn't known, rather than assert a specific vendor's behavior it has no way to check.

> [!IMPORTANT] Food for Thought
> If you were specifying this engine from scratch, which of the two triggers above would you make wait for drain, and which would you make abort immediately -- and would your answer change knowing that QP resets are usually deliberate and expected, while MR deregistrations racing a live transfer are usually a sign something upstream already went wrong?

**Where this leaves the reader.** This is the ninth Part in this book to reach a mid-transaction reset question, and the answer is the same shape every time: the problem is real, named precisely, and the resolution mechanics sit past the edge of what any public source documents. [How to Read This Guide](/overview/how-to-read-this-guide) named this as a recurring checklist item worth recognizing on purpose -- this chapter is that pattern's own honest conclusion for the one resource that turned out to have two independent triggers instead of one.

## Interface reference

| Trigger | Open question | This book's position |
|---|---|---|
| **QP RESET mid-burst** | does the transfer drain to completion, or abort immediately | unresolved -- both are physically plausible, no source confirms either |
| **MR deregistration mid-burst** | private per-burst translation copy, or a live re-touch of the shared table | unresolved -- inherits [Mid-Transaction Deregistration](/mr/mid-transaction-deregistration)'s own stated position exactly |
| **Both triggers racing the same transfer** | not addressed by any chapter in this book | named here as a real possibility, not analyzed further |
| **A transfer not yet started when either trigger fires** | none -- simply discarded | settled, no open question |

## References

- [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) and [Mid-Transaction Deregistration](/mr/mid-transaction-deregistration) -- both already-published chapters this one inherits its open questions from directly, rather than re-deriving or resolving further.
- No IBTA, IETF, or public vendor source documents mid-transaction DMA teardown behavior for either trigger -- this chapter states that gap plainly, consistent with every other Part's own mid-transaction chapter reaching the same limit.
