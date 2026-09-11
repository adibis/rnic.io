---
title: "Event and Multi-CQ Arbitration"
weight: 6
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "Multi-QP Error Arbitration named this Part as the place a real, documented storm scenario gets its general answer: many QPs forced to ERROR by one shared fault, all needing to report at once through what is, at that instant, one serialized path. This chapter is that answer -- and its quieter, steady-state counterpart on the notification side."
prev: /cq/cq-overflow
---

## Overview

[Multi-QP Error Arbitration](/qp/multi-qp-error-arbitration) already established a real, documented scenario as fact: when a Completion Queue or Shared Receive Queue itself hits an error, every QP associated with it transitions to ERROR and fires a QP-fatal event simultaneously, not one at a time. That chapter named the shape of the problem this Part would have to answer -- many simultaneous producers, one serialized reporting structure -- without answering it. This chapter is where it gets answered, alongside a related but distinct question: many Completion Queues, sharing one notification channel, competing for the same interrupt.

## Locator

![The master RNIC block diagram with both CQ Storage and Host: Event / Interrupt highlighted, every other block dimmed -- this chapter spans both the writing side and the notification side of arbitration.](/images/cq/06-event-and-multi-cq-arbitration-locator.svg)

Two different structures this Part has already covered both funnel into a shared resource under the right conditions. [CQE Generation](/cq/cqe-generation) is one serialized write path several QPs can converge on at once. [Creating a Completion Queue](/cq/creating-a-completion-queue)'s event channel and completion vector are one notification path several CQs can converge on too. This chapter covers what decides servicing order when either actually happens.

## Block diagram

![Two parallel arbitration questions. On the write side: three QPs forced into ERROR by one shared CQ or SRQ fault, all needing to flush through the same serialized CQE Generation path at once -- who goes first. On the notify side: three Completion Queues sharing one completion vector, competing for the same interrupt line -- which gets serviced first. Both funnel into the same three candidate policy shapes this book has already used twice: round-robin, weighted, and priority, with no public source confirming which a real RNIC actually runs for either question.](/images/cq/06-event-and-multi-cq-arbitration-block.svg)

## Deep walkthrough

**The write-side scenario, restated precisely from where it was left.** [Single-QP Error Handling](/qp/single-qp-error-handling) already named the trigger: a QP-fatal event caused by an error on a Completion Queue or Shared Receive Queue itself, rather than anything specific to one QP. When that happens, every QP associated with the faulted structure transitions to ERROR at the same instant, and [CQE Generation](/cq/cqe-generation) already established what each of those QPs then has to do: flush every remaining entry through as a CQE, in order, regardless of signal flags. The problem this chapter actually owns is what happens when *several* QPs need to do that at the same moment, through what is, at that instant, one shared, serialized write path. Something has to decide whose flush gets serviced first.

**The notify-side scenario -- a quieter version of the same shape.** [Creating a Completion Queue](/cq/creating-a-completion-queue) already named the mechanism: an optional completion vector steers which interrupt line a given CQ's notifications land on, and a device supporting more than one vector can spread different CQs across different lines specifically to reduce contention. That spreading is a real mitigation, not a complete answer -- a device only supports so many vectors, and an application can create far more CQs than that. Multiple CQs sharing one vector is the ordinary, expected case for any application with more Completion Queues than the hardware has interrupt lines to give them, not a misconfiguration. When two of those CQs both have something to report at nearly the same instant, the single interrupt line they share can only carry one signal at a time -- something has to decide which CQ's notification the host actually services first.

> [!NOTE] Concept
> Both scenarios in this chapter reduce to the identical shape: several sources need one shared, serialized resource at overlapping moments, and something has to pick an order. One is about writing a CQE; the other is about raising a notification. Neither is a capacity problem the way [CQ Overflow](/cq/cq-overflow) is -- there's no missing room here, only a moment where two or more things want the same single-servicing-at-a-time gate simultaneously.

**The same three candidate shapes, because it's the same underlying question.** [Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling) already worked through round-robin, weighted round-robin, and priority for a different shared gate -- whose WQE gets fetched next -- and [Eviction Policy](/qp/eviction-policy) worked through the same three shapes again for cache victim selection. Round-robin services every waiting QP-flush or CQ-notification in a fixed rotation, fair and starvation-free but blind to whether one of them is more urgent than the rest. Weighted round-robin fixes that blindness at the cost of needing something to own the weights. Strict priority serves whichever QP or CQ matters most predictably fast, at the structural starvation risk [Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling) already named for the exact same tradeoff. This book doesn't have a citation pinning down which of these a real RNIC actually runs for either the write-side or notify-side question here, the same honest position taken every other time this shape has come up.

**Why a QP-fatal storm specifically raises the stakes on getting this right.** An ordinary moment of two QPs' signaled completions landing close together is unremarkable -- [The Completion Queue](/cq/the-completion-queue) already established that a shared CQ's ring is a strict FIFO regardless of which QP fed it, and nothing about ordinary traffic requires a policy decision beyond "whoever's write actually arrives first." A shared-fault storm is different in kind: potentially dozens or hundreds of QPs (per [Multi-QP Error Arbitration](/qp/multi-qp-error-arbitration)'s own framing) all becoming eligible to flush at the exact same instant, not spread out over time the way ordinary traffic naturally is. Whatever policy governs this gate matters most precisely when it's under the most simultaneous demand -- the same lesson [CQ Overflow](/cq/cq-overflow)'s provisioning rule already drew from the identical event.

**What happens next.** Once a CQE actually gets written and, if applicable, a notification actually gets raised, both this Part's earlier chapters and this one are done with it -- *Mid-Transaction CQ Reset*, the last chapter in this Part, is what happens to a Completion Queue itself while it's still in this kind of active use and something tries to tear it down underneath that activity.

## References

- No IETF or IBTA standard governs which arbitration policy a real RNIC runs for either simultaneous CQE writes or simultaneous CQ notifications -- it is implementation-specific silicon and firmware design, not a wire-format contract any standards body defines. This chapter's treatment reasons from the same candidate-policy shapes already sourced and cited in [Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling) and [Eviction Policy](/qp/eviction-policy), not a new citation of its own.
