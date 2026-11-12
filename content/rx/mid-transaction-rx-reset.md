---
title: "Mid-Transaction RX Reset"
weight: 8
date: 2026-09-09
publishDate: 2026-09-09
draft: false
description: "Mid-Transaction TX Reset closed TX Pipeline asking what becomes of a WQE caught mid-build when RESET lands. This chapter closes RX Pipeline asking the receiving side's version of the same question -- and finds it sharper, because this part's own reassembly state can leave a partial write sitting in memory with nothing downstream aware it is incomplete."
prev: /rx/hardware-assisted-rx-dispatch-engines
metaDescription: "What a QP reset does to an RNIC's receive path mid-message: reassembly state that can leave a partial write in memory nothing knows is incomplete."
seoTitle: "Resetting the RX Path Mid-Transaction"
---

## Overview

RESET is legal from any state, including RTS with a packet actively occupying one of this part's own pipeline stages. What has to become of that packet -- does whatever this part was doing to it complete, get aborted, or land in some undefined state -- is exactly the kind of question [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) already flagged as underdocumented for a Queue Pair's other in-flight state. This chapter has one sharper version of that question to add, specific to this part alone: an open multi-packet message can leave a partial write already sitting in host memory, with nothing downstream of this pipeline aware that it stopped short of complete.

## Locator

![The master RNIC block diagram with RX Parser / Reassembly and RX PSN Check both highlighted, and QP Table reachable via the reset arrow -- this chapter's reasoning reaches into every stage this part covered, at the instant RESET lands on one of them.](/images/rx/08-mid-transaction-rx-reset-locator.svg)

[Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) closed [Queue Pair Management](/qp/) on a resource torn down while something was still actively using it. [Mid-Transaction TX Reset](/tx/mid-transaction-tx-reset) closed [TX Pipeline](/tx/) on the same shape of question for the sending side. This chapter closes [RX Pipeline](/rx/) on it for the receiving side: a packet mid-parse, mid-sequence-check, or mid-reassembly, at the exact instant its owning Queue Pair's state machine takes a RESET transition.

## Block diagram

![A single QP's packet at three possible RX pipeline positions when RESET lands -- mid-parse in RX Packet Parse, resolved at the PSN gate, or mid-reassembly with an open Multi-Packet Reassembly context. The first two dead-end in an open question; the mid-reassembly position carries the sharper one instead: a destination buffer in host memory already holding real bytes from the FIRST and MIDDLE packets, with no documented signal that the write stopped short of complete.](/images/rx/08-mid-transaction-rx-reset-block.svg)

## Deep walkthrough

**What actually has to be reconciled, specific to this part.** By the time RESET can land on a Queue Pair with receive traffic in flight, this part has established at least three places a packet or its effects might be sitting: partway through header parsing and validation ([RX Packet Parse](/rx/rx-packet-parse)); resolved and waiting at the sequence-number gate ([PSN Check and Out-of-Order Arrival](/rx/psn-check-and-out-of-order-arrival)); or, the position unique to this part, mid-message with an open reassembly context holding a partially-advanced destination address ([Multi-Packet Reassembly](/rx/multi-packet-reassembly)). RESET's job is to leave none of these positions holding state that corrupts whatever this Queue Pair does next -- but the specific sequencing that gets there for a real implementation is not something any public source this book found documents in general.

**The first two positions are underdocumented in the same way Queue Pair Management and TX Pipeline already found their own in-flight state to be.** A packet mid-parse when RESET lands has produced no committed effect yet -- nothing has been written to memory, no sequence number has been consumed -- which makes it the most straightforward of the three to reason about, though still not something any source states as settled fact: it is plausible that hardware simply stops treating the frame's bytes as belonging to this Queue Pair the instant RESET lands, discarding whatever partial parsing work was in progress. A packet resolved at the sequence-number gate but not yet handed off to placement sits in a similarly underdocumented middle state -- the check has already passed, but nothing has touched memory yet, so what "aborting" it actually requires is, at most, discarding a decision rather than unwinding a write.

**The third position is where this part's own contribution to the general problem actually lives.** An open reassembly context means bytes from this message's FIRST and however many MIDDLE packets are already sitting in host memory, correctly placed, with the destination buffer's remaining portion still holding whatever it held before this message started writing to it. If RESET lands here, the memory those earlier packets wrote to does not un-write itself -- there is no mechanism named anywhere in this book's sources that reverses a DMA placement already committed. What is left behind is a buffer holding an ambiguous mixture: real bytes this message actually sent, followed by bytes this message never got to send, with nothing in the buffer's own contents distinguishing where one ends and the other begins. Whether a real implementation surfaces any signal that this happened -- to the local application whose memory now holds a truncated write, or otherwise -- is not documented anywhere this book found, and this chapter does not invent an answer where none is sourced.

**Why this is a sharper version of the general problem, not just another instance of it.** [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) and [Mid-Transaction TX Reset](/tx/mid-transaction-tx-reset) both describe state that either was never actually committed anywhere durable, or that stays entirely inside the adapter's own structures once RESET reconciles it. This chapter's reassembly case is different in kind: the effect already escaped the adapter and landed in host memory before RESET had any chance to intervene, which means whatever RESET does to this part's own internal reassembly state, it cannot retroactively make the application's memory consistent again. The adapter can clear its own bookkeeping; it cannot un-write bytes that already left it.

> [!NOTE] Concept
> A reset can always clear what an adapter still holds. It cannot retract a DMA write that already landed in host memory -- once bytes leave the adapter, RESET's authority stops at the boundary they crossed.

**What this chapter does not claim.** This is not evidence that real hardware handles this badly, or a claim that a real implementation leaves applications exposed to silently truncated writes with no recourse at all -- it is entirely possible that a real design signals this condition through a mechanism this book's current sources simply do not document, or that operational practice elsewhere in the stack (an application-level completion accounting scheme, a checksum over the expected total) catches what this pipeline cannot. What this chapter can state with confidence is narrower: nothing in the public record confirms such a mechanism exists, and the architectural shape of the problem -- a partial write already committed to memory with no undo path -- is real regardless of whether any given implementation has built a way to signal it.

> [!IMPORTANT] Food for Thought
> If you were designing the signal for this condition, where would you put it -- a poisoned completion, a sentinel value in the buffer itself, something in the QP's error-event path? What would each choice cost, and what would it still fail to catch?

**Where this leaves the reader, and this part.** Three open positions, all underdocumented, one of them carrying a consequence that reaches outside the adapter's own state entirely. This chapter does not manufacture answers for any of them, and does not let the shape of the reassembly case above imply the other two are better understood than they are.

> [!TIP] Why
> Building hardware that could retroactively invalidate or annotate a partial write already placed in host memory would require treating every DMA placement as provisional until a message's LAST packet confirms it complete -- real complexity on a path this part has already established needs to sustain line rate, for a case that is by definition rare and hard to test exhaustively. Leaving a partial write in place, undone and unsignaled unless some other layer catches it, is cheaper to build and verify, at the cost of a real, if rare, window where an application's memory can hold bytes from a message that never actually finished. Whether any real vendor has judged that cost worth paying down, and built the signaling mechanism to close it, is exactly the kind of fact this book has not been able to confirm from public sources.

This closes [RX Pipeline](/rx/). [Reliability Engine](/reliability/) picks up directly from the sequence-number gate and out-of-order rejection this part named but deliberately did not resolve: what actually happens to a sender's outstanding work once a gap or a timeout reaches it.

## References

- No IETF or IBTA standard governs whether a specific implementation can abort or annotate a partially-placed multi-packet write on a QP reset -- that is implementation-specific silicon design. This chapter's treatment is reasoned architectural inference, stated as such, not grounded in a citable source.
