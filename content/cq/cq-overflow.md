---
title: "CQ Overflow"
weight: 5
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "The Replay Buffer can't safely evict, so hardware enforces admission control instead. A Completion Queue's overflow story is starker than that: the specification doesn't require hardware to protect against it at all. Provisioning correctly is a software responsibility, stated plainly as one."
prev: /cq/cqe-generation
---

## Overview

[Replay Buffer Capacity and Backpressure](/reliability/replay-buffer-capacity-and-backpressure) already established that some resources can't answer a capacity question by evicting something -- an unacknowledged replay entry is unsafe to sacrifice, so hardware enforces admission control instead, refusing to transmit past what its Replay Buffer can hold. A Completion Queue's version of this question has a different, more exposed answer: the specification doesn't require hardware to detect or prevent overflow at all. This isn't a gap this book is flagging as unknown -- it's a documented, deliberate design choice, and provisioning around it is a real software obligation stated as such.

## Locator

![The master RNIC block diagram with CQ Storage highlighted -- the master diagram inventory already names overflow logic as living inside this same block, not a separate top-level structure.](/images/cq/05-cq-overflow-locator.svg)

[The Completion Queue](/cq/the-completion-queue) already named the trigger: a producer index that's caught all the way up to the consumer index, with nowhere left to write a new entry. This chapter is what happens when that trigger actually fires.

## Block diagram

![Three QPs, each with its own maximum outstanding Work Request count, all pointing at one Completion Queue -- the sum of those three ceilings compared against the CQ's own depth. If the depth covers the full sum, behavior is deterministic even if every QP's outstanding work completed at once. If it doesn't, the specification does not require hardware to detect or protect against the result, and a QP flush is exactly the scenario that turns "every possible outstanding WR completing at once" from a hypothetical into a real event. A closing note states the one guarantee that does hold regardless: overflow of one CQ can't reach a QP or a CQ that isn't party to it.](/images/cq/05-cq-overflow-block.svg)

## Deep walkthrough

**The specification's own position, stated directly rather than inferred.** This is a real, documented design choice, not an open question this book is declining to answer: the RDMA interface is not required to perform CQ overflow detection or protection at all, which makes an overflow-specific error code optional rather than mandatory -- and when overflow does occur anyway, the result is left indeterminate. Nothing in [The CQE](/cq/the-cqe) or [The Completion Queue](/cq/the-completion-queue) changes because of this; it's a separate fact layered on top of both, about what happens at the one moment those chapters' structures run out of room.

> [!CAUTION] Gotcha
> Don't read "not required to detect or protect" as "never does." A specific implementation is free to add overflow detection beyond what the standard mandates. What the standard actually guarantees is narrower and worth being precise about: *if* an implementation doesn't detect it, that's compliant, and *if* overflow happens on such an implementation, no particular outcome is owed to software.

**The provisioning rule that keeps this from being a real problem in practice.** The same source that states overflow protection is optional also states exactly what makes overflow avoidable by construction: software should size a Completion Queue so that if every Work Request currently capable of being outstanding actually completed at once, there would still be room for all of their CQEs. Stated as a concrete inequality: for deterministic behavior, the sum of the maximum outstanding Work Request counts across every Queue Pair pointing at one CQ should never exceed that CQ's own depth. Get that provisioning right, and overflow simply can't happen, no matter how completions happen to be timed. Get it wrong, and the specification's silence on what happens next is the entire answer this book can honestly give.

**"Every possible outstanding WR completing at once" is not a hypothetical -- this book has already named the mechanism that does it.** [Single-QP Error Handling](/qp/single-qp-error-handling) already established that a QP forced to ERROR gets both its queues flushed, and [CQE Generation](/cq/cqe-generation) already named what that means for this Part specifically: a flush writes a CQE for every remaining entry, in order, regardless of any individual Work Request's own signal flag. A provisioning scheme that only accounts for the *steady-state* rate of signaled completions, and not for what a single QP's full outstanding count would dump into its CQ all at once during a flush, is exactly the gap the sizing rule above is warning against.

> [!TIP] Why
> The provisioning rule's phrasing -- "if every possible outstanding WR were to complete" -- is doing real work by including the word *possible* rather than *typical*. A CQ sized for ordinary, steady-state completion traffic can still overflow the instant one associated QP faults and flushes its entire outstanding count at once, precisely because a fault is exactly the scenario where many completions arrive together instead of spread out over time. Sizing against the worst case rather than the common case is the only way the deterministic guarantee actually holds when it's needed most -- during the one event most likely to produce a burst.

**The one guarantee that survives overflow regardless of provisioning.** Even granting that an overflowing CQ's own behavior is indeterminate, its blast radius is not left equally unbounded: overflow of one CQ must not affect a Queue Pair that doesn't report completions to it, and must not affect any other Completion Queue. Whatever indeterminate thing happens to this CQ and the QPs actually pointing at it, it stays contained to them -- the same shape of isolation boundary [Single-QP Error Handling](/qp/single-qp-error-handling) already drew around one QP's own fault, applied here to a shared structure's overflow instead.

**Why this Part doesn't get an admission-control gate the way Reliability Engine did.** [Replay Buffer Capacity and Backpressure](/reliability/replay-buffer-capacity-and-backpressure) argued that hardware has to gate transmission itself, because a lost replay entry breaks the reliability protocol's own ability to retransmit -- the protocol depends on that data surviving. A Completion Queue's contents aren't load-bearing for RC's wire-level reliability at all: an operation that overflows its CQ has still been correctly sent, acknowledged, and retried exactly as if its completion had been recorded -- what's lost is only software's bookkeeping of that fact, not the protocol's own correctness. That's the real reason the standard can afford to leave this to software provisioning discipline instead of mandating a hardware gate: getting it wrong costs software its own visibility, not the wire protocol its guarantees.

## References

- The RDMA Protocol Verbs Specification states directly that CQ overflow detection and protection are optional, that overflow behavior is otherwise indeterminate, that a Consumer should provision a CQ's depth against the sum of every associated QP's maximum outstanding Work Request count for deterministic behavior, and that overflow of one CQ must not affect QPs or CQs not party to it. These are software-interface-level guarantees, not IBTA-V1 wire-format requirements -- no IETF or IBTA standard mandates a specific hardware overflow-protection mechanism.
