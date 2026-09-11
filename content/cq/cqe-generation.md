---
title: "CQE Generation"
weight: 4
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "Nothing else in this Part is fed by both the TX and RX pipelines at once. This chapter is the convergence point -- and the one place a real, asymmetric rule decides whether a completion gets written at all: every receive-side event does, unconditionally, while a send-side event only does if something actually asked for it."
prev: /cq/creating-a-completion-queue
---

## Overview

Every chapter so far in this Part described something that already exists -- a CQE's fields, the ring holding it, the handshake that creates that ring. This chapter is about the moment one of those CQEs actually comes into being. It's the one block in this book fed by both the TX and RX pipelines at once, and the one place a real asymmetry decides whether a given event produces a CQE at all.

## Locator

![The master RNIC block diagram with CQE Gen highlighted -- fed by two separate arrows, a dashed one labeled "send completion (if signaled)" from TX Packet Build and a solid one labeled "receive completion" from RX PSN Check, converging into CQ Storage.](/images/cq/04-cqe-generation-locator.svg)

Every other chapter in this Part is scoped to one side of this diagram or the other -- the entry, the ring, the handshake that creates it. This block is different: it's the one point where the TX and RX pipelines this book has spent two entire Parts on both terminate in the same structure.

## Block diagram

![A Work Request completing on the TX side passes through a real gate -- is it signaled, either by its own flag or this QP's own signal-all default -- before a CQE gets written; an unsignaled completion produces nothing. A Receive Request being consumed on the RX side passes through no gate at all: every one produces a CQE, unconditionally. Both paths converge on a shared write into CQ Storage. A third, override path shows a QP forced to ERROR flushing every remaining entry through as a CQE regardless of its own signal flag.](/images/cq/04-cqe-generation-block.svg)

## Deep walkthrough

**The send side is gated; the receive side isn't -- confirmed, not inferred.** [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor) already named the `signal` flag as one of four requests a Work Request's flags field carries: a completion-request flag selecting whether this specific operation generates a completion at all. That flag only exists on the send side. Documented directly: completion generation is application-selectable per Work Request on the send side -- either by that Work Request's own flag, or unconditionally for every Work Request on a QP that was set up with a signal-all default at creation -- while a Receive Request carries no equivalent flag at all. Every posted Receive Request that gets consumed produces a completion, full stop, because the application needs to know a receive landed (to reuse or repost the buffer it named) in a way it doesn't necessarily need to know that an unsignaled SEND or WRITE landed.

> [!TIP] Why
> This isn't an oversight in the interface -- it's the same reasoning [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor) already gave for why the `signal` flag exists on the send side at all. A requester posting a long burst of ordinary WRITEs often only cares that the *last* one landed, not each individual one -- selective signaling lets it avoid paying a CQE, and a poll to consume it, for every single operation in between. A receive has no equivalent "I don't care about this one" case: the buffer a Receive Request named is only safe to reuse once software knows it's been consumed, so there's nothing to selectively suppress.

**What "gated" actually means -- a completion that's suppressed, not delayed.** An unsignaled Work Request that completes successfully doesn't produce a CQE later, or a smaller one, or one some other structure holds instead -- it produces nothing at all. Software's only way to learn that specific operation succeeded is indirectly: a later, signaled operation on the same Send Queue completing implies every earlier operation on that same queue also completed, since a Send Queue is consumed strictly in order ([WQE Prefetch](/tx/wqe-prefetch) already established this ordering). Selective signaling trades per-operation confirmation for fewer completions to generate and poll, not for a cheaper or deferred version of the same confirmation.

> [!CAUTION] Gotcha
> Don't read "unsignaled" as "unreliable." An unsignaled WRITE is still fully covered by RC's reliability contract -- [Reliability Engine](/reliability/) still requires an ACK and still retries it on loss, exactly as if it had been signaled. What's suppressed is only the *local* completion notification telling this side's software the operation succeeded, not the wire-level guarantee that it actually happened.

**The override: a flush ignores this entire gate.** [Single-QP Error Handling](/qp/single-qp-error-handling) already established what happens once a QP is forced to ERROR: hardware flushes both of its queues, "generating a real CQE for each entry, even though none of them succeeded." This chapter's contribution is naming what that means for the gate above specifically -- a flush-generated CQE is written regardless of whether the original Work Request was ever signaled at all. The signal flag answers "does a successful completion get reported"; it says nothing about a failed one, and a flush doesn't consult it.

**Which CQ a completion actually targets.** Nothing about generation decides, on the fly, where a given CQE goes -- that association is already settled well before this stage ever runs. [Installing a QP](/qp/installing-a-qp) already named a Completion Queue as one of the objects that has to exist, and be pointed to, before a QP is much use at all. CQE Generation consumes that already-established binding; it doesn't create or resolve it.

**Where this connects.** A written CQE advances the producer index [The Completion Queue](/cq/the-completion-queue) already covered, landing in whichever ring [Creating a Completion Queue](/cq/creating-a-completion-queue)'s handshake produced. If that CQ was created with an event channel, this is also the moment that would trigger a notification instead of leaving the entry for a polling loop to find -- what happens once many CQs are all wired up that way at once is this Part's own arbitration chapter, still ahead.

## References

- No IETF or IBTA standard governs the internal mechanics of CQE generation -- it is implementation-specific silicon design, not a wire-format contract any standards body defines. The signal-flag gating asymmetry between send-side and receive-side completions is grounded in public Work Completion and Work Request documentation, cited in-text above and in [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor), not a standards document.
