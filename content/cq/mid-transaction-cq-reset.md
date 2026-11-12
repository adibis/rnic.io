---
title: "Mid-Transaction CQ Reset"
weight: 7
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "Mid-Transaction TX Reset already gathered the citation this chapter needed before this Part even had a name: destroying a QP doesn't guarantee its CQEs get cleaned out of the Completion Queue it reported to. This closing chapter is what that leaves behind, and the one real dispute about whether a shared CQ's own cleanup path can be trusted not to touch a QP that never asked to be reset at all."
prev: /cq/event-and-multi-cq-arbitration
metaDescription: "Destroying a QP does not guarantee its CQEs are cleaned out of a shared Completion Queue. What that leaves behind, and whether CQ cleanup is safe."
seoTitle: "Resetting a Completion Queue Mid-Transaction"
---

## Overview

[Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) closed [Queue Pair Management](/qp/) by naming completion reporting as one of the things a QP reset has to reconcile, without resolving it -- flagging directly that a DMA write's outcome "might" still be watched for "if completion reporting... hasn't itself been told the QP is gone." This chapter is where that gets resolved, closing this Part the same way: what a torn-down QP leaves behind in a Completion Queue it no longer exists to report to.

## Locator

![The master RNIC block diagram with CQ Storage highlighted and every other block dimmed -- this chapter covers what a torn-down QP leaves behind in this structure.](/images/cq/07-mid-transaction-cq-reset-locator.svg)

Every earlier chapter in this Part assumed the QPs feeding a Completion Queue stay alive. This chapter is what happens when one of them doesn't.

## Block diagram

![Two sequences compared directly. Recommended: a QP is moved to ERROR, its CQEs are drained from a Completion Queue it shares with another QP, and only then is it destroyed -- leaving the shared CQ clean, with only the other QP's entry remaining. Skipped: the same QP is destroyed directly, without draining, leaving its CQEs still sitting in the shared ring -- and if that QP's number gets reused by a new QP later, the leftover entries read as if they belong to it. A closing note states the sourced IBTA-V1 practice this compares against, and the real, documented dispute over whether a shared CQ's own cleanup path protects the other QP's entry during this sequence.](/images/cq/07-mid-transaction-cq-reset-block.svg)

## Deep walkthrough

**The documented practice, cited directly rather than reasoned toward.** [Mid-Transaction TX Reset](/tx/mid-transaction-tx-reset) already gathered this citation while covering a different part of this same problem: IBTA-V1 (§10.2.4.4) states that it's good programming practice to move a QP to ERROR and retrieve its CQEs *before* destroying that QP, because destruction alone doesn't guarantee those CQEs get deallocated from the Completion Queue they were written into. That's this chapter's starting fact, not an inference -- the standard itself names the risk and the recommended practice in the same breath.

**Why "doesn't guarantee deallocation" matters specifically when a CQ is shared.** [The Completion Queue](/cq/the-completion-queue) already established that one CQ routinely serves many QPs at once, with no per-entry ownership beyond a field inside the CQE itself. A destroyed QP's undrained CQEs don't vanish just because the QP that produced them no longer exists -- they sit in that shared ring exactly where they were written, indistinguishable in kind from any other entry, until something polls them out.

> [!CAUTION] Gotcha
> This is the same shape of hazard [The QP Table](/qp/the-qp-table) already named for QP numbers themselves: a freed identifier, reissued to something new, with old state still capable of being misread as belonging to the new owner. There, the risk was a stale packet naming a reused QP number. Here, it's a leftover CQE, sitting in a *shared* structure that outlives the QP that wrote it, at risk of being read as if it belonged to whichever QP that same number gets reissued to next.

**The real, documented dispute this chapter doesn't get to resolve.** This isn't a hypothetical concern this book is raising on its own: public engineering discussion around at least one shipped implementation has disputed whether its completion-cleanup path, run during QP destruction, actually protects a shared Completion Queue correctly -- specifically, whether clearing risks touching entries that belong to a *different* QP sharing that same CQ, one that never asked to be reset or destroyed at all. [Mid-Transaction TX Reset](/tx/mid-transaction-tx-reset) already surfaced this as a forward-pointer to this Part; this chapter is where it actually lands, and it lands as an open question rather than a resolved one -- this book does not have a source confirming which shipped implementations get this right.

**What this means for the recommended sequence in practice.** Draining before destroying is the documented mitigation, but it only protects the CQEs belonging to the QP being destroyed -- it says nothing about whether the *destruction step itself*, once initiated, is guaranteed not to disturb neighboring entries in a shared CQ that belong to unrelated, still-active QPs. Software following the recommended practice correctly still depends on the implementation's own cleanup path behaving as the dispute above questions.

> [!IMPORTANT] Food for Thought
> If you were verifying a real implementation's QP-destruction path, how would you actually test whether it protects a shared Completion Queue's other entries correctly -- and would you trust a test that never happened to trigger the race, given how narrow the window between "drain completes" and "destruction runs" might be?

**The other direction: destroying the CQ itself while QPs still point at it.** Everything above is about a QP going away while the Completion Queue it reported to survives. [Creating a Completion Queue](/cq/creating-a-completion-queue) left the reverse case as this chapter's own subject: what has to be true of a CQ's *own* destruction while Queue Pairs are still associated with it. A real interface answers this one plainly, by refusing the request outright: a Completion Queue cannot be destroyed while any Queue Pair is still associated with it, and destruction fails until every associated QP has itself been destroyed (or reassigned to point elsewhere) first. This is a harder guarantee than the QP-side case above -- not a documented best practice software can choose to skip, but a precondition the interface enforces before it will act at all.

**Where this leaves the Part, and the book's own QP-reset story.** [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) posed its open question about a QP undergoing RESET specifically -- returning to INIT for reuse, not being destroyed. [Mid-Transaction TX Reset](/tx/mid-transaction-tx-reset) already drew that line carefully on the TX side: IBTA-V1 §10.2.4.4 governs QP *destruction*, not "a hardware-enforced precondition on the RESET transition," because "a QP being reset for reuse is a different act from a QP being destroyed outright." Everything this chapter has covered inherits that same scope -- it answers what a *destroyed* QP leaves behind in a Completion Queue, and what destroying a CQ itself requires. The RESET-for-reuse case [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) actually asked about -- whether completion reporting has been told a QP is gone, mid-reset, before something downstream keeps watching for its status -- is not the same question, and this book has not found a source that resolves it. This Part closes here without pretending a destroy-scoped answer also settles the reset-scoped one.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- IBTA-V1 §10.2.4.4 recommends the ERROR-then-drain-CQEs practice named above, for QP destruction specifically, not the RESET-for-reuse transition.
- No IETF or IBTA standard resolves whether a specific implementation's completion-cleanup path correctly isolates a shared Completion Queue's other entries during QP destruction -- that is implementation-specific silicon and driver design. This chapter's treatment of that open dispute is grounded in public engineering discussion, not a standards document.
- The CQ-destruction precondition (an interface refusing to destroy a Completion Queue while any Queue Pair remains associated with it) is grounded in public completion-queue lifecycle documentation, not a standards document -- no IETF or IBTA standard mandates this specific enforcement.
