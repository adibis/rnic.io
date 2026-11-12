---
title: "Mid-Transaction Rate-State Reset"
weight: 10
date: 2026-09-12
publishDate: 2026-09-12
draft: false
description: "Mid-Transaction QP Reset named this resource directly, before this Part had a title: congestion/rate-limiter state, built the same table-plus-working-set-plus-eviction way the Replay Buffer is, and left for whenever Congestion Control Engine got written. It's written now, and the honest answer is the same one that chapter already gave -- this book still doesn't know."
prev: /congestion/eviction-policy
metaDescription: "What happens to a QP's DCQCN rate-limiter state and its pending timers when the QP is reset with traffic in flight, and what no public source settles."
seoTitle: "Resetting Rate-Limiter State Mid-Transaction"
---

## Overview

[Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) named this Part's own resource among the five kinds of live state a QP's RESET has to reconcile, back when this Part had no chapters at all: "current injection rate, outstanding DCQCN feedback timers." That chapter was explicit that no public source documents the specific sequencing for any of its five, rate-limiter state included. This closing chapter inherits that position rather than resolving it further than the sources actually support.

## Locator

![The master RNIC block diagram with Rate Limiter highlighted and every other block dimmed -- this chapter covers what a RESET or destruction leaves behind in this structure, closing the same open question Mid-Transaction QP Reset already named.](/images/congestion/10-mid-transaction-rate-state-reset-locator.svg)

[Eviction Policy](/congestion/eviction-policy) covered what decides which entry leaves a working set, if one exists at all. This chapter is about what happens to an entry -- cached or not -- when the QP behind it is reset or destroyed while that entry is still live.

## Block diagram

![Three real races, none resolved by any public source. RESET issued with a rate cut in flight: does the in-progress reaction-factor update complete, or is it discarded mid-write? RESET issued with the recovery state machine mid-phase: does fast recovery or additive increase get to finish its current step? Destruction of a QP whose rate-limiter entry is cached: does the cache line get explicitly invalidated, or does it sit stale until naturally evicted? Each ends the same way -- a real implementation has to answer it, and this book does not have a source saying how.](/images/congestion/10-mid-transaction-rate-state-reset-block.svg)

## Deep walkthrough

**What's actually being asked, restated precisely.** [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) already framed the general shape this chapter's own version of the question takes: RESET is legal from RTS, with a QP's operations actively in flight, and IBTA's own contract for what RESET clears is thin. Applied to this Part's own resource specifically: a rate cut can be underway -- the reaction factor mid-update, the target rate just set from a freshly-arrived CNP -- at the exact instant RESET lands. So can the recovery state machine [The Rate Control State Machine](/congestion/the-rate-control-state-machine) covers, mid-phase, a fast-recovery or additive-increase step already begun but not yet committed.

> [!CAUTION] Gotcha
> This is not a hypothetical timing coincidence to wave away. A QP under active congestion is, by construction, one that's recently received a CNP or is mid-recovery from one -- exactly the QPs most likely to also be carrying other in-flight state RESET has to reconcile. A congested QP and a QP with a resettable race are not two independent populations; they overlap by construction.

**Three real races, stated as open rather than resolved.** First: RESET issued while a rate cut is mid-update -- does the reaction-factor write complete before RESET takes effect, or is it discarded in whatever state it was in? Second: RESET issued mid-recovery-phase -- does the current fast-recovery or additive-increase step finish, or does RESET simply invalidate the whole state machine's progress regardless of phase? Third, and structurally different from the first two: a QP destroyed (not merely reset) while its rate-limiter entry sits in an on-chip working set, if [Caching](/congestion/caching)'s open question resolves in favor of one existing at all -- does destruction explicitly invalidate that cached entry, or does it sit stale, structurally the same hazard [The QP Table](/qp/the-qp-table) already named for a freed-and-reissued QP number, until ordinary eviction pressure eventually reclaims it.

> [!IMPORTANT] Food for Thought
> Of the three races above, which would you expect a real implementation to actually get wrong first under test -- and would your answer change between a QP that resets and reconnects quickly versus one whose number sits quarantined for a while before reuse, the same open question [The QP Table](/qp/the-qp-table) already left unresolved for QP numbers generally?

**Why this book doesn't resolve these the way it resolved some of Completion Queue / Event Management's own version of this question.** [Mid-Transaction CQ Reset](/cq/mid-transaction-cq-reset) had a real, direct citation to work with -- IBTA-V1 §10.2.4.4's documented recommendation for CQE draining before QP destruction. This chapter has no equivalent. [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) already searched for one, for this exact resource, before this Part existed, and came back empty. Nothing this Part's own research turned up since changes that -- the DCQCN paper and its cross-checked second implementation both describe steady-state algorithm behavior in detail, and neither says anything about a reset or destruction transition arriving mid-cycle.

**What a freshly re-installed QP gets instead.** Whatever this chapter can't resolve about *cleanup*, [Enabling Congestion Control](/congestion/enabling-congestion-control) already resolved for *re-arming*: a QP re-entering INIT and proceeding back through RTS gets the same fixed starting point any newly-installed QP does -- full line rate, no slow start, reaction factor reset to 1. Whatever uncertainty this chapter leaves about how the *old* incarnation's state was actually torn down, the *new* incarnation's starting point is settled and sourced.

**Where this leaves the Part.** [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset)'s own open question about this resource has a partial answer now, the same shape [Mid-Transaction CQ Reset](/cq/mid-transaction-cq-reset) gave for its own resource: this chapter names the races precisely, confirms they remain genuinely unresolved by anything public, and refuses to manufacture a cleaner answer than the sources support. This Part closes here.

## References

- No IETF or IBTA standard, and no source this book found -- including the primary sources this Part draws on throughout -- documents rate-limiter state's behavior across a RESET or destruction transition. [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) already searched for this and found the same gap before this Part was written; this chapter confirms it remains a gap rather than re-searching from nothing.
