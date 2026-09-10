---
title: "Mid-Transaction TX Reset"
weight: 8
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "Mid-Transaction QP Reset closed Part 1 asking what happens to a DMA write already mid-burst when RESET lands. This chapter asks the TX-pipeline-specific version of the same question -- and unlike that chapter, it has a named, shipped RNIC on record as simply unable to answer it in hardware at all."
prev: /tx/hardware-assisted-tx-scheduling-engines
---

## Overview

RESET is legal from any state, including RTS with a WQE actively occupying one of this part's own pipeline stages. What has to become of that WQE -- does the build it's partway through complete, get aborted, or land in some undefined state -- is exactly the kind of question [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) already flagged as underdocumented for the QP's other in-flight state. This chapter isn't limited to that same reasoned-analysis-only territory, though: engineers who worked on software support for a named, shipped RNIC are on record describing it as unable to do this cleanly in hardware at all.

## Locator

![The master RNIC block diagram with WQE Fetch / TX Scheduler and TX Packet Build both highlighted, and QP Table reachable via the reset arrow -- this chapter's reasoning reaches into every stage this part covered, at the instant RESET lands on one of them.](/images/tx/08-mid-transaction-tx-reset-locator.svg)

[Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) closed Part 1 on a resource torn down while something was still actively using it -- a replay-buffer entry, a partially-reassembled message, a DMA write mid-burst. This chapter closes Part 3 on the same shape of question, asked specifically of everything this part just built: a WQE mid-fetch, mid-prefetch-buffer-residency, mid-build, or mid-schedule, at the exact instant its owning QP's state machine takes a RESET transition.

## Block diagram

![A single QP's WQE at four possible TX pipeline positions when RESET lands -- ring-resident, prefetch-buffer-resident, mid-build in TX Packet Build, or waiting at the TX Scheduler gate. Three positions dead-end in an open question; the mid-build position carries a real citation instead: HiP08 hardware cannot flush outstanding work in error state, and software must detect and flush it on the next operation.](/images/tx/08-mid-transaction-tx-reset-block.svg)

## Deep walkthrough

**What actually exists to reconcile, specific to this part.** By the time RESET can land on a QP with traffic in flight, this part has established at least four places a WQE might be sitting: still in the ring, untouched ([The Send Queue Ring](/tx/the-send-queue-ring)); resident in the on-chip prefetch buffer, read ahead of need but not yet processed ([WQE Prefetch](/tx/wqe-prefetch)); partway through header assembly ([TX Packet Build](/tx/tx-packet-build)); or queued at the scheduling gate, fetched and ready but waiting its turn behind other QPs' traffic ([Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling)). RESET's job is to leave none of these positions holding state that corrupts whatever gets built or scheduled next -- but the specific sequencing that gets there for each of the four isn't something any public source for a real RNIC documents in general.

**A named, shipped exception to "no public source documents this."** A software patch's commit description for Huawei's HiP08 RoCE hardware -- written by engineers who had to work around the limitation directly -- states plainly that the hardware itself "lacks ability to flush outstanding WQEs if QP state gets into errored mode." This is not this book's reasoned speculation about a hard problem -- it's a real, named, shipped-silicon admission from the people who had to build software around it: the hardware itself cannot cleanly flush a QP's outstanding work purely in response to an error/reset transition. The documented workaround is entirely software-side: detect the errored QP state during whatever operation happens to touch it next (a subsequent post-send, post-receive, or completion poll) and perform the flush from software at that point, deferred rather than run immediately to avoid doing that work while holding a lock protecting other pipeline state. This is directly relevant to this chapter's own open question: at least one real vendor's answer to "what happens to outstanding TX-side work on reset" is, in effect, *hardware doesn't fully answer it, software has to notice and clean up afterward* -- a different answer than "hardware handles it," and a citable one.

**A related, documented spec boundary, named without overclaiming -- and for a different act than this chapter's own.** IBTA-V1 (§10.2.4.4) names a related but distinct scenario: it's good programming practice to move a QP to ERROR and retrieve its CQEs *before destroying* that QP, since destruction alone doesn't guarantee the QP's CQEs get deallocated from a Completion Queue. That's a recommendation for QP teardown, not a hardware-enforced precondition on the RESET transition this chapter is actually about -- a QP being reset for reuse is a different act from a QP being destroyed outright, and IBTA doesn't gate this chapter's RESET-for-reuse case behind a mandatory drain step. Public engineering discussion around at least one shipped implementation has disputed whether a real adapter's completion-cleanup path adequately protects a shared Completion Queue during this destruction sequence, since clearing entries indiscriminately risks touching a different QP that never asked to be reset at all. This is a real, documented spec/implementation tension -- but it's precisely about Completion Queue cleanup during QP destruction, which is Part 7 (Completion Queue / Event Management)'s territory, not this part's WQE-side state during a RESET-for-reuse. Named here as an adjacent data point and a forward-pointer, not claimed as direct evidence about the TX pipeline's own four positions above.

**Where this leaves the reader.** One of this chapter's four open positions -- mid-build -- has a shipped, named answer, and it's an admission of a hardware gap rather than a documented guarantee. The other three -- ring-resident, prefetch-buffer-resident, and scheduler-queued -- remain exactly as underdocumented as [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) found its own five satellite structures to be. This chapter doesn't manufacture answers for the three it can't source, and doesn't let the one answer it found imply the other three are equally well understood.

> [!TIP] Why
> The HiP08 case above is more than a trivia item: it's evidence for which side of a genuine design tradeoff at least one real vendor landed on. Building hardware that can cleanly abort a WQE at any of this part's four pipeline positions, at any instant, is substantial silicon complexity -- every stage needs an abort path, not just a forward path, and that path has to be provably correct for a case (an error transition) that is by definition rare and hard to test exhaustively. Punting that complexity to software -- let the hardware simply stop being coherent about outstanding work on error, and have software notice and clean up on the next touch -- is cheaper to build and verify, at the cost of a window where outstanding state is inconsistent until software happens to look. HiP08's own documented history shows a vendor accepting that cost, at least for one hardware generation, rather than building the harder, fully-hardware-flushable alternative.

> [!IMPORTANT] Food for Thought
> HiP08 shows one vendor's engineering documentation admitting hardware can't flush outstanding work in error state, punting the problem to software instead. Did other vendors make the opposite trade and build a fully hardware-flushable pipeline? What would you actually check on real hardware to find out, given the public record doesn't say either way?

This chapter closes Part 3. Part 4 (RX Pipeline) runs a related set of questions against the receiving side of everything this part just built.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- IBTA-V1 §10.2.4.4 recommends the ERROR-then-drain-CQEs practice named above, for QP destruction specifically, not as a precondition on the RESET transition this chapter covers.
- No IETF or IBTA standard governs whether a specific RNIC's hardware can flush outstanding TX-side work on an error/reset transition -- that is implementation-specific silicon design. This chapter's treatment beyond the specified sequence is grounded in the sources cited in-text above, not a standards document.
