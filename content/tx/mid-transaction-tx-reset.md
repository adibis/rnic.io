---
title: "Mid-Transaction TX Reset"
weight: 8
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "Mid-Transaction QP Reset closed Part 1 asking what happens to a DMA write already mid-burst when RESET lands. This chapter asks the TX-pipeline-specific version of the same question -- and unlike that chapter, it has a real, named, shipped RNIC on record as simply unable to answer it in hardware at all."
prev: /tx/hardware-assisted-tx-scheduling-engines
---

## Locator

![The master RNIC block diagram with WQE Fetch / TX Scheduler and TX Packet Build both highlighted, and QP Table reachable via the reset arrow -- this chapter's reasoning reaches into every stage this Part covered, at the instant RESET lands on one of them.](/images/tx/08-mid-transaction-tx-reset-locator.svg)

[Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) closed Part 1 on a resource torn down while something was still actively using it -- a replay-buffer entry, a partially-reassembled message, a DMA write mid-burst. This chapter closes Part 3 on the same shape of question, asked specifically of everything this Part just built: a WQE mid-fetch, mid-prefetch-buffer-residency, mid-build, or mid-schedule, at the exact instant its owning QP's state machine takes a RESET transition.

## Overview

RESET is legal from any state, including RTS with a WQE actively occupying one of this Part's own pipeline stages. What has to become of that WQE -- does the build it's partway through complete, get aborted, or land in some undefined state -- is exactly the kind of question [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) already flagged as underdocumented for the QP's other in-flight state. This chapter isn't limited to that same reasoned-analysis-only territory, though: a real, shipped RNIC is on record, in its own kernel driver's commit history, as genuinely unable to do this cleanly in hardware at all.

## Block diagram

<!-- IMAGE: A single QP's WQE shown at four possible pipeline positions simultaneously (ring/not-yet-fetched, prefetch buffer, mid-build in TX Packet Build, waiting at the TX Scheduler gate), each position connected by a dashed line to a large "RESET" arrow striking down from outside, mirroring qp/mid-transaction-qp-reset.md's visual convention exactly. Each dashed line ends in a red question mark except the "mid-build" position, which instead carries a small citation flag reading "HiP08: hardware cannot flush this in error state -- driver must detect and flush on next operation (real, shipped limitation)." A caption at the bottom reads "four positions, one real answer and three open questions -- consistent with this chapter's own evidence." -->

## Deep walkthrough

**What actually exists to reconcile, specific to this Part.** By the time RESET can land on a QP with traffic in flight, this Part has established at least four places a WQE might be sitting: still in the ring, untouched ([The Send Queue Ring](/tx/the-send-queue-ring)); resident in the on-chip prefetch buffer, read ahead of need but not yet processed ([WQE Prefetch](/tx/wqe-prefetch)); partway through header assembly ([TX Packet Build](/tx/tx-packet-build)); or queued at the scheduling gate, fetched and ready but waiting its turn behind other QPs' traffic ([Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling)). RESET's job is to leave none of these positions holding state that corrupts whatever gets built or scheduled next -- but the specific sequencing that gets there for each of the four isn't something any public source for a real RNIC documents in general.

**A real, named, shipped exception to "no public source documents this."** A Linux kernel patch series for Huawei's HiP08 RoCE hardware (the `RDMA/hns` driver) states directly, in its own commit description, that "HiP08 RoCE hardware lacks ability to flush outstanding WQEs if QP state gets into errored mode." This is not this book's reasoned speculation about a hard problem -- it's a named vendor's own driver source admitting a genuine hardware limitation on real, shipped silicon: the hardware itself cannot cleanly flush a QP's outstanding work purely in response to an error/reset transition. The driver's own workaround is entirely software-side: detect the errored QP state during whatever operation happens to touch it next (a subsequent post-send, post-receive, or CQE poll) and perform the flush from software at that point, deferred through a workqueue specifically to avoid sleeping while holding a spinlock -- itself a real bug an earlier version of the same patch series had to fix. This is directly relevant to this chapter's own open question: at least one real vendor's answer to "what happens to outstanding TX-side work on reset" is, in effect, *hardware doesn't fully answer it, software has to notice and clean up afterward* -- a genuinely different answer than "hardware handles it," and a real, citable one.

**A related dispute, worth citing precisely rather than overclaiming.** A `linux-rdma` mailing-list thread documents IBTA-V1's specified sequence for this exact boundary -- transition to ERROR, drain every CQE associated with that QP, only then transition to RESET (§10.2.4.4/§10.3.1) -- and a named maintainer (Tom Talpey) directly disputing whether Mellanox's mlx5 driver actually follows it: a function that clears CQEs from a *shared* Completion Queue on one QP's reset, called out as "a bug" specifically because a CQ is commonly shared across multiple QPs, and clearing its entries indiscriminately risks touching a different QP that never asked to be reset at all. This is a real, citable instance of a spec/implementation gap at exactly this boundary -- but it's precisely about Completion Queue cleanup, which is Part 7 (Completion Queue / Event Management)'s territory, not this Part's WQE-side state. Named here as a real, adjacent data point and a forward-pointer, not claimed as direct evidence about the TX pipeline's own four positions above.

**Where this leaves the reader.** One of this chapter's four open positions -- mid-build -- has a real, shipped, named answer, and it's an admission of a hardware gap rather than a documented guarantee. The other three -- ring-resident, prefetch-buffer-resident, and scheduler-queued -- remain exactly as underdocumented as [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) found its own five satellite structures to be. This chapter doesn't manufacture answers for the three it can't source, and doesn't let the one real answer it found imply the other three are equally well understood.

> **Design rationale.** The HiP08 case above is worth reading as more than a trivia item: it's real evidence for which side of a genuine design tradeoff at least one real vendor landed on. Building hardware that can cleanly abort a WQE at any of this Part's four pipeline positions, at any instant, is real silicon complexity -- every stage needs an abort path, not just a forward path, and that path has to be provably correct for a case (an error transition) that is by definition rare and hard to test exhaustively. Punting that complexity to software -- let the hardware simply stop being coherent about outstanding work on error, and have the driver notice and clean up on the next touch -- is cheaper to build and verify, at the cost of a real window where outstanding state is inconsistent until software happens to look. HiP08's driver history shows a real vendor accepting that cost, at least for one hardware generation, rather than building the harder, fully-hardware-flushable alternative. Whether other vendors made the opposite trade is exactly the kind of fact this book has not been able to confirm from public sources.

## Interface / timing reference

| Decision point | What a real design has to choose | This book's position |
|---|---|---|
| WQE still ring-resident, untouched, on RESET | Simply invalid on next read vs. explicit invalidation | Not documented; reasoned as low-risk since it was never fetched |
| WQE in prefetch buffer on RESET | Discarded vs. allowed to complete prefetch harmlessly | Not documented |
| WQE mid-build (TX Packet Build) on RESET | Abort in hardware vs. cannot abort, software must detect and clean up | **Real, shipped example of the latter**: HiP08 RoCE hardware, confirmed via kernel patch series |
| WQE queued at the TX Scheduler gate on RESET | Removed from scheduling consideration vs. serviced anyway | Not documented |
| Related: CQE cleanup on a shared CQ during QP reset | Per-QP-only vs. whole-CQ clearing | Real, disputed on a linux-rdma mailing-list thread against IBTA-V1 §10.2.4.4/§10.3.1 -- Part 7's territory, named here as an adjacent, real data point |

This closes Part 3. Eight chapters, one TX pipeline, the same checklist [How to Read This Guide](/overview/how-to-read-this-guide) previewed, applied to a datapath instead of a resource -- two items dropped because they genuinely didn't apply, one merge, and one chapter (TX Packet Build) that exists entirely outside the checklist because a resource-management checklist was never built to ask about a pipeline's own transform stage. Part 4 (RX Pipeline) runs a related set of questions against the receiving side of everything this Part just built.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications) -- IBTA-V1 §10.2.4.4/§10.3.1 specifies the ERROR-drain-then-RESET sequence named above.
- No IETF or IBTA standard governs whether a specific RNIC's hardware can flush outstanding TX-side work on an error/reset transition -- that is implementation-specific silicon design. This chapter's treatment beyond the specified sequence is grounded in the sources cited in-text above, not a standards document.
