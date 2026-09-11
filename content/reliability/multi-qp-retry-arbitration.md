---
title: "Multi-QP Retry Arbitration"
weight: 6
date: 2026-09-09
publishDate: 2026-09-09
draft: false
description: "One QP's retransmission is a self-contained event. Many QPs losing packets to the same root cause at the same moment is a different problem entirely -- and on a shared fabric, that correlation is the common case, not the exception."
prev: /reliability/go-back-n-retransmission
next: /reliability/replay-buffer-capacity-and-backpressure
---

## Overview

[Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling) already established that many QPs with ready work compete for one shared transmit pipeline, and that something has to decide whose work gets serviced next. A retransmission is, mechanically, just another packet asking for that same pipeline -- but it arrives under a circumstance ordinary traffic doesn't share: a timeout already happened, meaning this QP has already paid a measurable delay once, and every other QP retrying at the same moment likely has too, often because they share whatever caused the original loss in the first place.

## Locator

![The master RNIC block diagram with the Retry Timer and TX Packet Build/Scheduler blocks both highlighted -- many QPs' retransmissions all compete for the same physical send pipeline Multi-QP TX Scheduling already covers for ordinary traffic.](/images/reliability/06-multi-qp-retry-arbitration-locator.svg)

[Go-Back-N Retransmission](/reliability/go-back-n-retransmission) covered one QP's retransmission in isolation. This chapter asks what happens when many QPs need that same service from the same shared pipeline at once.

## Block diagram

![Several QPs' timers expiring in a tight window -- correlated by a shared upstream cause such as a congested link or a switch buffer event -- all presenting retransmission work to the same TX Scheduler at once, with a callout naming the new question this scenario raises: does retransmission get its own precedence, or does it compete on equal footing with fresh traffic.](/images/reliability/06-multi-qp-retry-arbitration-block.svg)

## Deep walkthrough

**Why this correlates instead of scattering randomly.** [Multi-QP Error Arbitration](/qp/multi-qp-error-arbitration) already established, for a different failure mode, that a single shared upstream cause can take out many QPs at effectively the same instant rather than one at a time. Packet loss has exactly the same correlation risk on a shared fabric: a congested link, a switch buffer overflow, or a brief outage doesn't pick on one QP's traffic specifically -- it drops packets indiscriminately from whatever happened to be crossing that link at that moment, which on any fabric carrying more than a handful of connections means several QPs losing packets from the same root cause within the same narrow window. Their timers, set against similar timeout values, expire close together as a direct consequence. Multi-QP retransmission demand is not an unlikely edge case this book is reasoning about for completeness; it's the expected shape of what happens after any shared-path loss event.

**The question ordinary scheduling never had to answer.** [Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling) covers round-robin, weighted, and priority approaches to servicing many QPs' ready sends fairly. Retransmission adds a question none of those approaches needed to consider: should a retransmission compete for the pipeline on the same footing as a QP's fresh, never-before-sent traffic, or does the fact that this data already failed once and already cost a full timeout argue for servicing it ahead of work that hasn't failed yet? Neither answer is free. Treating retransmissions identically to fresh traffic means a QP recovering from loss waits in the same queue as every QP that never had a problem, extending a delay that already happened once. Giving retransmissions precedence risks the opposite failure: a correlated loss event that hits many QPs at once could flood the pipeline with retransmission traffic ahead of everything else, starving QPs that have fresh work ready and never experienced the loss themselves.

**What a real implementation has to weigh, without a documented answer to cite.** No source this book found states which of these two positions -- equal-footing or priority-for-retransmission -- a specific shipped RNIC actually takes, and this chapter doesn't invent one. What can be reasoned honestly is the shape of the tradeoff: retransmission traffic is, by construction, traffic that already missed its first deadline, which argues for urgency: the request behind it has already waited [Retry and Timeout State](/reliability/retry-and-timeout-state)'s full timeout duration once, with the application still blocked waiting for it. But retransmission traffic arriving in a correlated burst is also exactly the traffic most likely to overwhelm a pipeline that's already demonstrated it's under stress, which argues for caution rather than urgency. A design has to pick a position on this spectrum, and this book can name the tradeoff without asserting which position any real chip has taken.

> [!IMPORTANT] Food for Thought
> Should a retransmission compete for the shared pipeline on equal footing with fresh traffic, or jump ahead of it because it already missed one deadline? Argue both sides before you decide which fabric characteristics would tip you one way or the other.

**What happens next.** This chapter closed the question of who gets serviced and in what order. What's still unresolved is capacity itself -- not who goes first, but what happens when the Replay Buffer this part depends on runs out of room entirely. [Replay Buffer Capacity and Backpressure](/reliability/replay-buffer-capacity-and-backpressure) is that chapter, immediately next.

> [!TIP] Why
> A design that gives retransmissions strict priority over fresh traffic accepts a real risk in exchange for faster recovery: a widespread loss event could turn a brief network problem into a longer one, as a flood of retransmissions crowds out the fresh sends that would otherwise be making progress on unaffected connections. A design that treats retransmissions as ordinary traffic accepts slower individual recovery -- a QP that already waited out a timeout waits again, behind whatever else is ready -- in exchange for never letting a correlated failure compound itself through retransmission traffic alone. Neither choice is obviously correct independent of the fabric's own loss characteristics: a fabric where correlated loss events are common and severe has a real argument for caution; one where loss is rare and isolated per-QP has less to lose from prioritizing recovery.

## References

- No IETF or IBTA standard governs how a real RNIC arbitrates retransmission traffic against fresh traffic across many Queue Pairs -- it is implementation-specific silicon and firmware design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is reasoned architectural inference, stated as such, not a citation.
