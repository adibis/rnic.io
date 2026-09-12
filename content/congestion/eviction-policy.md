---
title: "Eviction Policy"
weight: 9
date: 2026-09-12
publishDate: 2026-09-12
draft: false
description: "Every other eviction chapter in this book starts from a confirmed working set with confirmed pressure. This one starts from Caching's own open question -- and finds that if a working-set cache does exist here, the same three familiar candidates apply, fixed or programmable, exactly the way they already have three times before."
prev: /congestion/multi-flow-cnp-arbitration
next: /congestion/mid-transaction-rate-state-reset
---

## Overview

[Eviction Policy](/qp/eviction-policy), [Key Cache Eviction Policy](/mr/key-cache-eviction-policy), and [Replay Buffer Capacity and Backpressure](/reliability/replay-buffer-capacity-and-backpressure) each opened from a confirmed premise: a real, sourced working set, under real, sourced pressure. [Caching](/congestion/caching), this Part's own version of that chapter, couldn't offer the same premise -- whether rate-limiter state needs a working-set cache at all is still open. This chapter covers what eviction would look like if it does, and folds in the hardware-versus-software question [Hardware-Assisted Eviction Engines](/qp/hardware-assisted-eviction-engines) asks separately for its own resource, the same way [Replay Buffer Capacity and Backpressure](/reliability/replay-buffer-capacity-and-backpressure) already combined both questions into one chapter for the Replay Buffer.

## Locator

![The master RNIC block diagram with Rate Limiter highlighted and every other block dimmed -- this chapter covers what happens if this structure needs to reclaim a resident entry, conditional on Caching's own open question.](/images/congestion/09-eviction-policy-locator.svg)

[Caching](/congestion/caching) left open whether an on-chip working set exists here at all. This chapter is what decides which entry leaves, if one ever has to.

## Block diagram

![A conditional branch, not a confirmed pipeline: if Caching's shape B applies (every QP's entry resident, no working set), this chapter has no subject -- nothing is ever evicted because nothing is ever pressured. If shape A applies (a bounded working set backing a larger table), the same three candidate policies this book has already used twice -- round-robin, weighted, priority -- apply again, with the same fixed-versus-programmable question Hardware-Assisted Eviction Engines already asked for QP context.](/images/congestion/09-eviction-policy-block.svg)

## Deep walkthrough

**This chapter's entire premise is conditional, and that has to be said plainly before anything else.** [Caching](/congestion/caching) laid out two possibilities without a source to settle between them: either rate-limiter state is small enough for a real implementation to keep every QP's entry resident on-chip permanently, in which case nothing is ever pressured and this chapter has no subject at all, or it needs the same working-set/backing-store split [QP Caching](/qp/qp-caching) and [Key Caching](/mr/key-caching) already established for their own resources, in which case eviction is a real question with a real answer to give. Everything below assumes the second case, the same way [Caching](/congestion/caching) itself reasoned through it -- clearly labeled as conditional, not restated as settled fact.

> [!CAUTION] Gotcha
> Don't read the rest of this chapter as confirming that rate-limiter state gets evicted. It describes what eviction *would* look like, if [Caching](/congestion/caching)'s open question resolves the way [QP Caching](/qp/qp-caching)'s and [Key Caching](/mr/key-caching)'s already did. No source this book found confirms it does.

**If a working set exists, the same three candidates apply again.** [Eviction Policy](/qp/eviction-policy) already worked through the three shapes a real cache's victim-selection logic can take: round-robin, blind to how urgently any one entry is needed; weighted round-robin, fixed by giving some entries priority at the cost of needing an external weight-owner; and strict priority, predictably fast for whatever's marked most urgent, at a structural starvation risk for everything else. Nothing about rate-limiter state changes which of these three shapes is available -- the same tradeoffs [Eviction Policy](/qp/eviction-policy) already reasoned through for QP context apply unchanged here, if there's a working set to apply them to at all.

> [!WARNING] Analogy
> This is the same three-shape menu [Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling) and [Event and Multi-CQ Arbitration](/cq/event-and-multi-cq-arbitration) both reused for their own, differently-shaped scheduling questions. Round-robin, weighted, priority isn't a coincidence this book keeps rediscovering -- it's the same three answers to "who goes first, or who leaves first" recurring everywhere a shared resource has to be arbitrated or reclaimed, regardless of which resource it happens to be.

**A rate-limiter-specific wrinkle, worth naming even conditionally: what does evicting *this* entry actually cost?** [Eviction Policy](/qp/eviction-policy) already established that evicting a QP context costs a future cache miss -- a PCIe round trip, not a correctness problem, since the QP Table still holds the authoritative copy. The same would hold here, by the same reasoning [The Rate Limiter Table](/congestion/the-rate-limiter-table) already gave: this resource's own backing store, if one exists, is the authoritative copy, so evicting a cached rate-limiter entry costs a refetch, not lost state. But refetching isn't quite free of consequence the way a QP context refetch is: a rate limiter that's momentarily lost its live current-rate and reaction-factor tracking has to resume *from* the refetched value, not recompute what it would have been had it stayed resident -- a QP evicted mid-recovery and later reinstated resumes wherever its last-written state left off, potentially stale relative to how long it was actually away.

> [!IMPORTANT] Food for Thought
> If eviction does apply here, a rate-limiter entry that's evicted and later reinstated resumes from a possibly-stale rate rather than one reflecting how much time has actually passed. Would you treat that staleness as harmless -- the next CNP or recovery cycle corrects it soon enough -- or would you want eviction policy for this specific resource to weigh recency differently than it does for QP context, where no such staleness exists?

**Hardware-assisted or software-driven -- the same question, folded in rather than split out.** [Hardware-Assisted Eviction Engines](/qp/hardware-assisted-eviction-engines) asked, for QP context, whether victim selection runs as fixed silicon, a programmable engine, or a software trap -- and found real, dated evidence that at least one shipped RNIC sits at the software-driven end of that spectrum. Nothing about rate-limiter state changes that question's shape, and this Part doesn't have independent evidence pointing to a different answer for this specific resource -- so this chapter folds the question in rather than manufacturing a second chapter's worth of content where the honest answer is "the same open question, unresolved here too."

**Where this connects.** Whether any of this chapter actually applies depends entirely on [Caching](/congestion/caching)'s own open question. What happens to this state -- evicted or resident, cached or not -- when the QP itself is reset or destroyed mid-transaction is [Mid-Transaction Rate-State Reset](/congestion/mid-transaction-rate-state-reset), closing this Part.

## References

- No IETF or IBTA standard, and no source this book found, confirms rate-limiter state has an eviction policy at all -- this chapter's content is conditional on [Caching](/congestion/caching)'s own open question, reasoning by analogy to [Eviction Policy](/qp/eviction-policy) and [Hardware-Assisted Eviction Engines](/qp/hardware-assisted-eviction-engines)'s already-sourced treatment of QP context, not a new citation for this resource specifically.
