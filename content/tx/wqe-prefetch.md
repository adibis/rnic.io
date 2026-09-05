---
title: "WQE Prefetch"
weight: 3
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "Every caching chapter so far in this book resolves a key or a QP number that could arrive in any order. A Send Queue ring doesn't work that way -- it's consumed strictly in order, front to back -- and that one difference is why this chapter's on-chip structure is a prefetch buffer, not a lookup cache, even though it's solving the exact same PCIe-latency problem."
prev: /tx/the-send-queue-ring
next: /tx/tx-packet-build
---

## Locator

![The master RNIC block diagram with WQE Fetch / TX Scheduler highlighted -- this chapter covers the on-chip structure that holds WQEs read ahead of when the pipeline actually needs them.](/images/tx/03-wqe-prefetch-locator.svg)

[QP Caching](/qp/qp-caching) and [Key Caching](/mr/key-caching) both cover a demand-driven working set: a key or QP number arrives, possibly out of order, and the cache either holds it or doesn't. A Send Queue ring has no equivalent of "an arbitrary index arrives" -- it's consumed strictly in order, one WQE after the next, which is exactly why this chapter's structure is shaped differently from both of them.

## Overview

Fetching a WQE from host DRAM on every single doorbell would put a PCIe round trip on the critical path of every operation, the same problem [QP Caching](/qp/qp-caching) and [Key Caching](/mr/key-caching) both solve for their own resources -- but a Send Queue ring's strict, front-to-back consumption order means the adapter doesn't need to guess which WQE might be needed next the way a demand-driven cache does; it can simply read ahead of the consumer index before the pipeline actually asks. A real, shipped RNIC does exactly this: a dedicated on-chip WQE cache holding prefetched send WQEs, distinct from the QP-context and address-translation caches this book has already covered.

## Block diagram

![A Send Queue ring with a consumer index pointing at one slot and the next several slots ahead of it bracketed as a prefetched, on-chip lookahead window, arrowing down into a WQE Cache box in the same amber on-chip-SRAM family as QP Cache and Key Cache but with no hit/miss branch -- beside it, a dashed box noting that prefetch distance and trigger aren't publicly documented.](/images/tx/03-wqe-prefetch-block.svg)

## Deep walkthrough

**What actually exists -- a real, direct citation, not an inference from the caching pattern elsewhere in this book.** Kong et al. (NSDI 2023), describing a real RNIC's on-chip microarchitecture resources directly, name a WQE Cache alongside the QP-context cache and the MTT/MPT cache this book already covers, with a plain description of its job: "the Work Queue Entry (WQE) cache stores prefetched send WQEs and posted receive WQEs." This is the load-bearing fact for this chapter: a real, shipped RNIC does maintain a distinct on-chip structure specifically for WQEs read ahead of need, not a mechanism this book is inferring by analogy from the QP/key caching chapters.

**Where checked, and the honest limit of what's public.** The same paper does not give the WQE cache its own capacity figure the way it does for the shared 2MB SRAM figure this book's Part 1/2 caching chapters already cite -- it names the structure's existence and its job, not its size. It also doesn't state how many WQEs ahead of the consumer index get prefetched, or what triggers a prefetch (a fixed lookahead distance, decided once at design time, versus something adaptive to how quickly the pipeline is actually consuming). `[SPARSE PUBLIC SOURCING beyond the one sentence above]` -- neither the uRDMA paper nor the QP-scaling loss-recovery paper this book cites elsewhere mentions WQE prefetching at all, checked directly rather than assumed. This chapter names the real anchor plainly and doesn't invent the specifics no source confirms.

**Why this isn't a hit/miss structure the way QP Cache and Key Cache are.** Both of this book's earlier caching chapters describe a genuine miss path: a key or QP number the cache doesn't currently hold, requiring a fallback fetch and a measurable penalty. A WQE prefetch buffer's failure mode is different in kind -- there is no "wrong" WQE to look up, because the pipeline only ever wants the next one in strict ring order. What can actually go wrong here is a **prefetch that hasn't caught up**: the pipeline reaches the consumer index faster than the prefetch mechanism has read ahead, and the next WQE simply isn't on-chip yet when it's needed. That's a real stall, not a cache miss in the QP-Cache/Key-Cache sense -- and [Single-QP Pipeline Stalls](/tx/single-qp-pipeline-stalls) is where this chapter's failure mode actually gets treated in depth, rather than re-deriving it here.

**What happens next.** Once a WQE is on-chip -- prefetched ahead of need, or fetched just-in-time if the prefetch hasn't caught up -- it's ready for the stage that actually turns it into wire bytes: [TX Packet Build](/tx/tx-packet-build), covered next.

> **Design rationale.** A demand-driven cache and a lookahead prefetch buffer are both answers to the same underlying question -- how to avoid paying a PCIe round trip on every single operation -- but they're the right answer to genuinely different access patterns. QP numbers and keys arrive in whatever order a workload's traffic happens to produce, which is exactly why [QP Caching](/qp/qp-caching) and [Key Caching](/mr/key-caching) need real eviction policies deciding what to keep resident under pressure. A Send Queue's WQEs never arrive out of order relative to their own ring -- the next one needed is always, by construction, the next one in the ring -- so there's no victim to select and no eviction policy to design; the only real design question is how far ahead to read, a tuning parameter rather than a policy question. Building a WQE prefetch buffer as a lookahead structure rather than forcing it through the same hit/miss/eviction framing as this book's other caches isn't a missed opportunity for consistency -- it's matching the mechanism to an access pattern that's structurally simpler than the ones QP Cache and Key Cache actually have to handle.

## Interface / timing reference

| Property | Value | Source / notes |
|---|---|---|
| WQE cache -- existence and job | confirmed real | "stores prefetched send WQEs and posted receive WQEs" (Kong et al., NSDI 2023) |
| WQE cache -- capacity | not documented | no source gives this a specific figure, unlike the shared 2MB QPC/MTT figure cited elsewhere |
| Prefetch distance (WQEs read ahead of consumer index) | not documented | `[SPARSE PUBLIC SOURCING]` -- reasoned as a tuning parameter, not confirmed |
| Prefetch trigger (fixed vs. adaptive) | not documented | `[SPARSE PUBLIC SOURCING]` |
| Failure mode when prefetch hasn't caught up | a stall, not a cache miss | see [Single-QP Pipeline Stalls](/tx/single-qp-pipeline-stalls) |

[TX Packet Build](/tx/tx-packet-build) is next: the stage that takes an on-chip WQE and actually produces the wire packet [What Is RoCEv2](/overview/what-is-rocev2) already described field by field.

## References

- No IETF or IBTA standard governs WQE prefetch buffer sizing, depth, or trigger mechanics -- it is implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above, not a standards document.
