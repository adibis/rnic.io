---
title: "Single-QP Rate-State Faults"
weight: 7
date: 2026-09-12
publishDate: 2026-09-12
draft: false
description: "A malformed Congestion Notification Packet isn't a new failure mode this Part has to invent a story for -- it's an ordinary malformed packet, caught the same way any other one is. What's genuinely open is quieter: what a rate limiter's own gate is actually for, once the QP behind it can no longer send anything at all."
prev: /congestion/the-rate-control-state-machine
next: /congestion/multi-flow-cnp-arbitration
---

## Overview

Every other chapter in this Part assumed correct operation: a real Congestion Notification Packet, correctly formed, arriving for a QP that exists and is still sending. This chapter asks what happens when either of those assumptions fails for exactly one QP -- and finds that most of it isn't a new problem this Part has to solve on its own.

## Locator

![The master RNIC block diagram with Rate Limiter highlighted and every other block dimmed -- this chapter covers what happens to one QP's own entry when something goes wrong, not a separate fault-handling structure.](/images/congestion/07-single-qp-rate-state-faults-locator.svg)

[The Rate Control State Machine](/congestion/the-rate-control-state-machine) covered correct-case recovery. This chapter is what happens when a CNP isn't correct, or the QP it targets isn't in a state to use it.

## Block diagram

![Two failure paths, neither owned by this Part. An unexpected PSN or an unallocated DestQP is rejected by the same packet validation every other malformed packet already goes through, owned by RX Pipeline and The QP Table; a corrupted opcode is a genuine, unfilled gap in that same general coverage, not something this chapter can close on congestion control's behalf. A QP already in ERROR still receives CNPs addressed to it, but its rate-limiter gate has nothing left to protect, since Single-QP Error Handling's own flush has already stopped this QP from sending anything a rate limiter could pace.](/images/congestion/07-single-qp-rate-state-faults-block.svg)

## Deep walkthrough

**Most of a malformed CNP is not a new problem -- it's an old one, already solved.** [CNP Generation](/congestion/cnp-generation) named CNP's exact wire format: a specific opcode, a forced PSN of zero, a `DestQP` naming the flow being throttled. Two of the ways that shape can arrive violated already have a real, citable answer: an unexpected PSN and a `DestQP` that doesn't correspond to any QP this adapter currently has allocated are exactly the failure modes [RX Pipeline](/rx/)'s PSN-check stage and [The QP Table](/qp/the-qp-table)'s unallocated-index handling already cover for any packet, CNP or otherwise. A corrupted or unrecognized opcode is narrower: this book's own operation catalog doesn't specify what happens to a packet carrying one, for a CNP or any other packet type -- a real gap this chapter isn't in a position to close on congestion control's behalf, since it isn't this Part's mechanism to begin with.

> [!CAUTION] Gotcha
> Don't go looking for a congestion-control-specific packet-validation chapter for the PSN and DestQP cases -- there isn't one, on purpose, since a general answer already exists and already covers them. The opcode case is different: it's a genuine, unfilled gap in this book's own RX-side coverage, not a congestion-control-specific one this chapter is declining to solve out of restraint.

**A CNP for a real QP that's already in ERROR -- addressed, but with nothing left to protect.** [Single-QP Error Handling](/qp/single-qp-error-handling) already established that a QP in ERROR flushes every queued Work Request and stops sending anything new at all. A CNP can still arrive naming that QP's number -- nothing about the wire or the sender knows the destination has faulted -- and nothing about receiving it is itself invalid. But the rate limiter's entire purpose is pacing *future* transmission, and a QP in ERROR has none coming. The CNP's rate cut and reaction-factor update, if applied at all, would have no QP behavior left to actually pace.

> [!IMPORTANT] Food for Thought
> No source this book found says whether a real implementation bothers to apply the rate cut anyway, to a QP that's already stopped sending, or simply drops a CNP addressed to a faulted QP as a cheap no-op. Both are defensible: applying it costs nothing observable and keeps the logic uniform; skipping it saves a write to state nothing will ever read again before the QP is destroyed or reset. Which would you build, and would your answer change if this rate-limiter entry might outlive the fault, waiting to be reused by a future QP at the same number?

**What this chapter is not.** It isn't [Mid-Transaction Rate-State Reset](/congestion/mid-transaction-rate-state-reset)'s subject -- that chapter, closing this Part, covers what happens to this state specifically when a QP transition (RESET, or destruction) is issued while rate-limiter state is live, the same shape [Mid-Transaction QP Reset](/qp/mid-transaction-qp-reset) already asked of QP context generally. This chapter is narrower: a single QP's rate-limiter entry encountering a fault condition while the QP itself is otherwise untouched by a reset or teardown request.

**Where this connects.** Both paths this chapter named terminate outside this Part: malformed-packet rejection is [RX Pipeline](/rx/)'s general mechanism, and what ERROR does to a QP's outstanding work at all is [Single-QP Error Handling](/qp/single-qp-error-handling)'s. This chapter's own contribution is naming that neither needs a congestion-control-specific variant -- what's genuinely this Part's own open question is the narrower one above: whether a rate cut addressed to an already-faulted QP is applied or discarded.

## References

- No IETF or IBTA standard, and no source this book found, addresses malformed-CNP rejection or rate-cut handling for an already-faulted QP as congestion-control-specific concerns -- this chapter's position is that neither needs to be, reasoning by cross-reference to [RX Pipeline](/rx/)'s and [Single-QP Error Handling](/qp/single-qp-error-handling)'s already-published, general mechanisms rather than a new citation.
