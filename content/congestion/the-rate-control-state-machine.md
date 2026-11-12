---
title: "The Rate Control State Machine"
weight: 6
date: 2026-09-12
publishDate: 2026-09-12
draft: false
description: "A CNP cuts the current rate once, immediately. Getting it back is not one action but three -- a fast phase, a steady phase, and an aggressive phase, picked automatically by two counters this book already named, not chosen by whichever chapter happens to be reading them."
prev: /congestion/cnp-generation
next: /congestion/single-qp-rate-state-faults
metaDescription: "How DCQCN recovers after a CNP cut: fast recovery, additive increase and hyper increase, chosen by a timer count and a byte-counter count."
seoTitle: "DCQCN Rate Control State Machine"
---

## Overview

[Rate-Limiter Context](/congestion/rate-limiter-context) named the timer, byte counter, and their stage counts without saying what they were for. This chapter is what reads them: a real, three-phase state machine that decides, on every timer or byte-counter expiry, whether this QP's current rate should leap toward its target, creep toward it, or ramp aggressively past the caution either of those two phases would otherwise apply.

## Locator

![The master RNIC block diagram with Rate Limiter highlighted and every other block dimmed -- this chapter covers the logic that reads one QP's own stage counts, not a separate structure.](/images/congestion/06-the-rate-control-state-machine-locator.svg)

[CNP Generation](/congestion/cnp-generation) covered what triggers a rate cut. This chapter is what happens on every cycle after one -- the machinery that decides how fast recovery actually proceeds.

## Block diagram

{{< flow "rate-recovery" >}}

## Deep walkthrough

**The reset: one event, two immediate effects.** [CNP Generation](/congestion/cnp-generation)'s output is this chapter's own trigger. The instant a CNP arrives for this QP, two things happen at once, sourced directly from the algorithm this Part has used throughout: the current rate is cut -- remembered as the new target, then reduced by an amount the reaction factor scales -- and both the timer's and the byte counter's own stage counts reset to zero. Everything from this chapter onward describes what happens in the absence of another CNP, starting fresh from that reset point.

> [!CAUTION] Gotcha
> The reaction factor scaling the cut, and the stage counts driving recovery afterward, are two different pieces of state answering two different questions. One decides *how hard* this cut lands; the other decides *how the climb back* proceeds. A QP that's been cut hard and a QP that's been cut gently run through the exact same three recovery phases below -- the reaction factor doesn't pick a different recovery path, only a different starting point on the same one.

**Two independent triggers, two independent stage counts.** [Rate-Limiter Context](/congestion/rate-limiter-context) already named the asymmetry worth repeating here: a timer and a byte counter both run at once, and either firing advances only its own stage count, on its own schedule. A QP sending a high rate of small messages can advance its byte-counter stage well ahead of its timer-based stage; the reverse holds for a QP sending large, infrequent bursts. Neither stage count on its own decides anything -- it's the *comparison* between them, described next, that actually selects a phase.

**Phase one: fast recovery.** While the *larger* of the two stage counts is still below a fixed threshold -- meaning both counts are still low, since neither trigger has fired many times yet -- the current rate leaps directly toward the target: each expiry sets the current rate to the midpoint between itself and the target, roughly halving the remaining distance every step. One documented deployment fixes this threshold at five steps; a second, independent implementation defaults to three -- the same evidence this Part has already used elsewhere that these specific numbers are tuned parameters, not fixed constants.

> [!CAUTION] Gotcha
> This is checked against the *larger* stage count, not the smaller one -- both counts have to still be low for fast recovery to apply. A QP sending small messages fast can advance its byte-counter stage well past the threshold while its timer-based stage is still low; checking only the smaller count would keep that QP in fast recovery based on the trigger that's barely fired, ignoring the one that's already fired many times. The state machine cares whether *either* trigger has accumulated enough firings to signal the burst is over, not whether *both* have.

**Phase two: additive increase.** Once one stage count has crossed the threshold but the other hasn't -- exactly the straddling case the Gotcha above describes -- the climb changes character: instead of chasing an already-fixed target, the target itself starts climbing, by a fixed step each cycle, with the current rate continuing to close half the distance to it. This is the steady-state phase -- slower, more cautious, and where a QP spends most of its time recovering after the initial fast-recovery burst has closed most of the gap.

> [!TIP] Why
> Two phases, not one, because the two halves of recovery have opposite priorities. Immediately after a cut, the current rate is likely still far below what the link can actually sustain -- fast recovery's job is closing that gap quickly, accepting a coarser step size because there's a lot of ground to cover and low risk of overshooting a target that hasn't moved. Once the current rate is already close to the target, the same coarse step size would risk oscillating past a ceiling the network only just confirmed it could sustain -- additive increase's smaller, target-advancing steps trade recovery speed for stability once stability is the thing actually worth optimizing for.

**Phase three: hyper increase.** If both stage counts run well past the fast-recovery threshold with no new CNP arriving -- a long enough stretch of apparently uncongested sending -- a third, more aggressive phase engages, ramping harder than additive increase's fixed step allows. The algorithm this book sources throughout this Part defers this phase's own formula to a separate, underlying specification rather than restating it in full, and this book does the same rather than inventing one. What is documented directly: at least one real, named vendor found this phase, as originally specified, ineffective at actually delivering a fast ramp-up, and replaced it in shipped hardware with a different aggressive-increase scheme of their own -- rate increase that doubles in magnitude with each further congestion-free period, precisely so a QP that's been idle-of-congestion for a while reclaims its full share of bandwidth quickly rather than crawling back at additive increase's steady, deliberately-cautious pace.

> [!WARNING] Analogy
> Three phases picked automatically by a threshold comparison is the same shape [Single-QP Error Handling](/qp/single-qp-error-handling)'s own fault-classification logic uses -- a QP doesn't choose which recovery behavior applies, the state it's already in does. Here, the state is a pair of counters advancing since the last CNP; there, it's a specific fault condition. Neither is a decision this book's own reasoning makes on a chapter's behalf.

**Where this connects.** Every phase above only ever runs in the absence of a new CNP -- [CNP Generation](/congestion/cnp-generation)'s own mechanism can interrupt this state machine at any point, cutting the rate and resetting both stage counts regardless of which phase was active. What happens if this QP's rate-limiter context itself faults or holds a corrupted value mid-cycle is [Single-QP Rate-State Faults](/congestion/single-qp-rate-state-faults), next.

## Interface reference

| Phase | Trigger condition | Effect | Source |
|---|---|---|---|
| **Rate cut** | CNP arrives | current rate cut, scaled by reaction factor; both stage counts reset to 0 | DCQCN paper |
| **Fast recovery** | larger stage count still below threshold `F` (both counts low) | current rate set to midpoint of itself and target | DCQCN paper (`F`=5); a second, independent implementation defaults to 3 |
| **Additive increase** | one stage count past `F`, the other not yet, no new CNP | target rate climbs by a fixed step; current rate closes half the remaining distance | DCQCN paper |
| **Hyper increase** | smaller stage count also past `F` (both counts past it), no new CNP for an extended period | an aggressive ramp beyond additive increase's fixed step | formula deferred to an underlying spec by the sourced algorithm itself; at least one real vendor replaced this phase entirely in shipped hardware |

## References

- Zhu, Eran, Firestone, Guo, Lipshteyn, Liron, Padhye, Raindel, Haj Yahia, Zhang, "Congestion Control for Large-Scale RDMA Deployments," SIGCOMM 2015 -- the full rate-cut equations, the fast-recovery/additive-increase formulas, and the three-phase state machine (their own Figure 7), including the `F`=5 default and the hyper-increase phase's deferral to QCN's own specification.
- Voloshin, "Introduction to Congestion Control for RoCE," Broadcom white paper NCC-WP103 (2023) -- a second, independent, real implementation confirming the same three-phase shape under its own defaults (three fast-recovery steps, not five) and documenting its own replacement for the hyper-increase phase.
