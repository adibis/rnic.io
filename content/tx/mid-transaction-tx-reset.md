---
title: "Mid-Transaction TX Reset"
weight: 8
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "Mid-Transaction QP Reset closed Part 1 asking what happens to a DMA write already mid-burst when RESET lands. This chapter asks the TX-pipeline-specific version of the same question -- and unlike that chapter, it has a named, shipped RNIC on record as simply unable to answer it in hardware at all."
prev: /tx/hardware-assisted-tx-scheduling-engines
---

## Locator

![The master RNIC block diagram with WQE Fetch / TX Scheduler and TX Packet Build both highlighted, and QP Table reachable via the reset arrow -- this chapter's reasoning reaches into every stage this Part covered, at the instant RESET lands on one of them.](/images/tx/08-mid-transaction-tx-reset-locator.svg)

[Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) closed Part 1 on a resource torn down while something was still actively using it -- a replay-buffer entry, a partially-reassembled message, a DMA write mid-burst. This chapter closes Part 3 on the same shape of question, asked specifically of everything this Part just built: a WQE mid-fetch, mid-prefetch-buffer-residency, mid-build, or mid-schedule, at the exact instant its owning QP's state machine takes a RESET transition.

## Overview

RESET is legal from any state, including RTS with a WQE actively occupying one of this Part's own pipeline stages. What has to become of that WQE -- does the build it's partway through complete, get aborted, or land in some undefined state -- is exactly the kind of question [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) already flagged as underdocumented for the QP's other in-flight state. This chapter isn't limited to that same reasoned-analysis-only territory, though: a named, shipped RNIC is on record, in its own vendor's public engineering documentation, as unable to do this cleanly in hardware at all.

## Block diagram

![A single QP's WQE at four possible TX pipeline positions when RESET lands -- ring-resident, prefetch-buffer-resident, mid-build in TX Packet Build, or waiting at the TX Scheduler gate. Three positions dead-end in an open question; the mid-build position carries a real citation instead: HiP08 hardware cannot flush outstanding work in error state, and software must detect and flush it on the next operation.](/images/tx/08-mid-transaction-tx-reset-block.svg)

## Deep walkthrough

**What actually exists to reconcile, specific to this Part.** By the time RESET can land on a QP with traffic in flight, this Part has established at least four places a WQE might be sitting: still in the ring, untouched ([The Send Queue Ring](/tx/the-send-queue-ring)); resident in the on-chip prefetch buffer, read ahead of need but not yet processed ([WQE Prefetch](/tx/wqe-prefetch)); partway through header assembly ([TX Packet Build](/tx/tx-packet-build)); or queued at the scheduling gate, fetched and ready but waiting its turn behind other QPs' traffic ([Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling)). RESET's job is to leave none of these positions holding state that corrupts whatever gets built or scheduled next -- but the specific sequencing that gets there for each of the four isn't something any public source for a real RNIC documents in general.

**A named, shipped exception to "no public source documents this."** Publicly available engineering documentation for at least one shipped RNIC -- Huawei's HiP08 RoCE hardware -- states directly that the hardware itself "lacks ability to flush outstanding WQEs if QP state gets into errored mode." This is not this book's reasoned speculation about a hard problem -- it's a named vendor's own engineering documentation admitting a genuine hardware limitation on shipped silicon: the hardware itself cannot cleanly flush a QP's outstanding work purely in response to an error/reset transition. The documented workaround is entirely software-side: detect the errored QP state during whatever operation happens to touch it next (a subsequent post-send, post-receive, or completion poll) and perform the flush from software at that point, deferred rather than run immediately to avoid doing that work while holding a lock protecting other pipeline state. This is directly relevant to this chapter's own open question: at least one real vendor's answer to "what happens to outstanding TX-side work on reset" is, in effect, *hardware doesn't fully answer it, software has to notice and clean up afterward* -- a different answer than "hardware handles it," and a citable one.

**A related, documented spec boundary, named without overclaiming.** IBTA-V1 specifies a precise sequence for this exact boundary -- transition to ERROR, drain every CQE associated with that QP, only then transition to RESET (§10.2.4.4/§10.3.1). Public engineering discussion around at least one shipped implementation has disputed whether a real adapter's completion-cleanup path actually honors that sequence when a Completion Queue is shared across multiple QPs, since clearing entries indiscriminately on one QP's reset risks touching a different QP that never asked to be reset at all. This is a real, documented instance of a spec/implementation tension at exactly this boundary -- but it's precisely about Completion Queue cleanup, which is Part 7 (Completion Queue / Event Management)'s territory, not this Part's WQE-side state. Named here as an adjacent data point and a forward-pointer, not claimed as direct evidence about the TX pipeline's own four positions above.

**Where this leaves the reader.** One of this chapter's four open positions -- mid-build -- has a shipped, named answer, and it's an admission of a hardware gap rather than a documented guarantee. The other three -- ring-resident, prefetch-buffer-resident, and scheduler-queued -- remain exactly as underdocumented as [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) found its own five satellite structures to be. This chapter doesn't manufacture answers for the three it can't source, and doesn't let the one answer it found imply the other three are equally well understood.

> [!TIP] Why
> The HiP08 case above is more than a trivia item: it's evidence for which side of a genuine design tradeoff at least one real vendor landed on. Building hardware that can cleanly abort a WQE at any of this Part's four pipeline positions, at any instant, is substantial silicon complexity -- every stage needs an abort path, not just a forward path, and that path has to be provably correct for a case (an error transition) that is by definition rare and hard to test exhaustively. Punting that complexity to software -- let the hardware simply stop being coherent about outstanding work on error, and have software notice and clean up on the next touch -- is cheaper to build and verify, at the cost of a window where outstanding state is inconsistent until software happens to look. HiP08's own documented history shows a vendor accepting that cost, at least for one hardware generation, rather than building the harder, fully-hardware-flushable alternative.

> [!IMPORTANT] Food for Thought
> HiP08 shows one vendor's engineering documentation admitting hardware can't flush outstanding work in error state, punting the problem to software instead. Did other vendors make the opposite trade and build a fully hardware-flushable pipeline? What would you actually check on real hardware to find out, given the public record doesn't say either way?

This closes Part 3. Eight chapters, one TX pipeline, the same checklist [How to Read This Guide](/overview/how-to-read-this-guide) previewed, applied to a datapath instead of a resource -- two items dropped because they didn't apply, one merge, and one chapter (TX Packet Build) that exists entirely outside the checklist because a resource-management checklist was never built to ask about a pipeline's own transform stage. Part 4 (RX Pipeline) runs a related set of questions against the receiving side of everything this Part just built.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- IBTA-V1 §10.2.4.4/§10.3.1 specifies the ERROR-drain-then-RESET sequence named above.
- No IETF or IBTA standard governs whether a specific RNIC's hardware can flush outstanding TX-side work on an error/reset transition -- that is implementation-specific silicon design. This chapter's treatment beyond the specified sequence is grounded in the sources cited in-text above, not a standards document.
