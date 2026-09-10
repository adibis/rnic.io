---
title: "How to Read This Guide"
weight: 6
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "The same ten-item checklist gets applied to a different hardware resource four times in this book. Naming it once here, before Part 1 shows the fullest version of it, saves you from re-discovering it by coincidence four parts later."
prev: /overview/one-transaction-start-to-finish
---

[The previous article](../one-transaction-start-to-finish) walked one RDMA Write through the whole machine and closed on something structural, not incidental: the same storage-structure-plus-working-set shape showed up more than once in a single transaction, attached to two different hardware resources on two different machines. That wasn't an accident of which example operation got chosen. It's structural. This article names the pattern directly and tells you where in the book it recurs, so you recognize it on purpose from here on rather than noticing it again by coincidence in Part 5 (Reliability Engine).

## The checklist

Several parts of this book ask the same ten questions about a different piece of hardware state. In order: what does one instance of this resource look like, field by field. What structure holds every instance of it. What working set of those instances gets cached close to the datapath, and what happens on a miss. What does the software-to-hardware handoff look like when a new instance is created. Does the resource have a state machine hardware enforces transitions through. What happens when one instance faults, in isolation. What happens when many instances fault at once, and in what order hardware services or reports them. What policy decides which cached instance gets evicted when the working set is full. Does hardware offer a programmable or fixed engine to run that policy, or does software have to drive it directly. And last: what happens to an instance mid-operation, with state in flight, if it gets torn down or reset before that operation finished.

Ten questions, one resource. **[Part 1](/qp/), Queue Pair Management, is this checklist run against the Queue Pair, in full** — QP Context, then the QP Table, then QP Caching, then Installing a QP, then the QP State Machine, then Single-QP Error Handling, then Multi-QP Error Arbitration, then Eviction Policy, then Hardware-Assisted Eviction Engines, then Mid-Transaction QP Reset. Ten chapters, ten questions, in the same order given above. It's the longest, most granular part in this book, and not because the Queue Pair is more complicated than everything else here -- it's where the checklist gets shown in full once, so later parts don't have to re-derive it from scratch.

They don't. [Part 2](/mr/) runs the same checklist against the Memory Region and its Key Table — the same shape, one chapter short, because a Memory Region's lifecycle doesn't carry the state-machine richness a Queue Pair's does, and Part 2 says so explicitly rather than silently skipping a step. Part 5's replay buffer is a third instance of the same table-plus-working-set-plus-eviction question, this time applied to unacknowledged, in-flight packet data instead of connection state or address translations. Part 6 (Congestion Control Engine)'s per-QP rate-limiter state is a fourth. By the time you reach Part 6, you'll have seen this exact shape often enough that its chapter list should be predictable before you read it.

Knowing this changes how you should read the book. When Part 5 says "eviction policy" and cross-references Part 1 instead of re-explaining what an eviction policy is, that's not a shortcut taken to save the author's time — it's the guide telling you these are the same question, so an answer worked out once in Part 1 should transfer.

## The chapter template

Every chapter in Parts 1 and onward — not the [Part 0](/overview/) (Overview) chapters you've been reading, including this one — follows the same core shape, in this order:

- **Locator.** The master RNIC block diagram, with this chapter's block highlighted and everything else dimmed. It's both a "you are here" and the chapter's hero image, the same drawing every time, never a new diagram.
- **Overview.** One paragraph stating what this block does and why it exists, before any mechanism is described.
- **Block diagram.** A second diagram, zoomed to just this stage or structure, replacing the locator's whole-chip context with actual internal detail.
- **Deep walkthrough.** What a field or signal is, where in the pipeline it's checked, what failure mode it produces when it's wrong, and what happens next — retried, retracted, or aborted, at what scope. Where another chapter owns the next step, this section names that chapter and stops, rather than re-explaining what it already covers.
- **Design rationale sidebar.** Set visually apart from the rest of the chapter, and the one place this book allows itself a narrative voice: why this structure got built this way, and what got traded off to get there.

This is a shape to expect, not a checklist every chapter is forced through -- a chapter that has nothing real to say in a given section skips it rather than filling it with restated prose or an invented number to look complete. Once you've read a few chapters, you'll recognize the shape of the rest, even where one of them departs from it for a real reason.

Part 1, chapter 1 — *QP Context* — is where that shape starts, and where this book's real depth begins.

## References

This chapter is organizational, not technical -- it names a pattern in how this book is structured, not a wire format or hardware behavior, so no standards document applies. Every other chapter closes with one.
