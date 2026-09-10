---
title: "Single-QP Pipeline Stalls"
weight: 5
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "A large RDMA operation and a small, latency-sensitive one sharing the same TX pipeline sound like a scheduling question. A measured study says it's actually simpler and worse than that: the pipeline serves whatever is ready to send, in the order it became ready, and a fast RDMA write behind a slow one measurably collapses -- before any scheduling policy even gets a turn to decide otherwise."
prev: /tx/tx-packet-build
next: /tx/multi-qp-tx-scheduling
---

## Overview

A QP's WQE can be sitting on-chip, fully prefetched, and still not be ready to become a packet -- waiting on a resolved `rkey`, an SGE's permission check, or its own payload's DMA read to complete. A direct measurement of exactly this shape of problem exists, and it's worse than "this one QP waits a little longer": a real RNIC's finite internal queue slots mean one QP's large, slow-moving operation can measurably starve a different QP's small, otherwise-fast one, sharing the same physical pipeline resources, well before any scheduling policy has a chance to intervene.

## Locator

![The master RNIC block diagram with WQE Fetch / TX Scheduler and TX Packet Build both highlighted -- this chapter covers what stalls one QP's path through either stage, before the harder multi-QP question next chapter takes up.](/images/tx/05-single-qp-pipeline-stalls-locator.svg)

[TX Packet Build](../tx-packet-build) named what a built packet needs and left open what happens when one of those inputs isn't ready. This chapter is that question, for one QP in isolation -- what actually stalls its send path, and whether that stall touches anything beyond the QP that caused it.

## Block diagram

![QP A's small latency-sensitive WQE and QP B's large elephant-flow WQE both feeding one narrow shared gate representing finite internal queue slots not allocated per-QP -- QP B's large WQE occupies the gate while QP A's WQE stalls behind it, dashed and marked stalled, illustrating work-conserving head-of-line blocking confirmed to measurably collapse a stalled flow's throughput on real hardware.](/images/tx/05-single-qp-pipeline-stalls-block.svg)

## Deep walkthrough

**The measured shape of the problem, from a direct study.** Zhang et al. ("Justitia," arXiv 1905.04437), studying performance isolation across QPs sharing one real RNIC directly, found that a throughput-sensitive flow "experiences a continuous falloff in performance with the increasing number of background bandwidth-sensitive flows" sharing the same adapter -- a severe collapse on the hardware they tested, not a marginal one. This isn't a theoretical worst case reasoned about in the abstract -- it's a confirmed effect on real RNIC hardware, for exactly the shape of question this chapter is about: one QP's traffic pattern degrading a different QP's traffic, sharing nothing except the physical pipeline underneath both of them. The specific magnitude is that paper's own tested configuration, not a number this chapter treats as a general architectural constant -- what matters architecturally is that the collapse is severe enough to demand an answer, which is exactly what motivates the next two chapters.

**Why it happens, stated directly by the same source.** The paper's own diagnosis ties the effect to a scarce on-chip resource on that 2019-era hardware, not a policy choice: "the limited number of queues inside the RNIC hardware" is what drives head-of-line blocking of small flows behind large ones. The mechanism is straightforward once stated: a real RNIC's TX pipeline processes whatever's ready to send in a work-conserving way -- it doesn't sit idle waiting for a "more important" WQE to arrive if a large one is already ready -- and a large operation occupying the pipeline's finite internal resources (the same scarce queue-slot/buffer budget [QP Caching](/qp/qp-caching) and [Key Caching](/mr/key-caching) already established gets contended across every resource type sharing the on-chip SRAM) simply isn't preemptible mid-flight by a smaller, later-arriving one. A QP doesn't need to be doing anything wrong for this to happen to it -- it only needs to be sharing a pipeline with a QP that is.

**Where this is checked, and the honest limit of what's public.** No source found describes the *specific* internal pipeline stage where a stalled QP's WQE actually waits -- whether it's held at the fetch stage (Part 3's own WQE Fetch/Scheduler block), at the packet-build stage the previous chapter covered, or at an SGE-resolution step consulting the Key Table. `[SPARSE PUBLIC SOURCING]` on the exact stage; described architecturally here rather than invented, the same honest treatment [Eviction Policy](/qp/eviction-policy) gave a similarly thin corner in Part 1. What the measurement above does confirm with confidence is that *some* real stage in a real RNIC's pipeline exhibits this behavior on shipped hardware -- this isn't a hypothetical risk this book is reasoning its way toward.

**Isolation -- or the lack of it, measured rather than assumed.** [Single-QP Error Handling](/qp/single-qp-error-handling) established a genuine isolation guarantee for one QP's *fault* -- a QP in ERROR cannot touch a sibling QP's state. This chapter's finding is the opposite shape of result for ordinary, non-faulting traffic sharing a pipeline: nothing about a large but entirely legitimate operation on one QP is isolated from a different QP's throughput, because both are drawing on the same finite, shared pipeline resource by construction.

> [!CAUTION] Gotcha
> Don't carry Part 1's fault-isolation guarantee forward into a context it was never meant to cover. Fault isolation and throughput isolation are different guarantees -- one holds here, the other, measurably, often doesn't without an explicit scheduling policy intervening.

**What happens next.** Whether -- and how -- a real design can prevent one QP's legitimate traffic from stalling another's is exactly [Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling)'s subject, next: the same measured anomaly this chapter names is the concrete motivating problem that chapter's scheduling policies exist to address.

> [!TIP] Why
> A work-conserving pipeline -- one that never idles while there's any ready work, regardless of whose work it is or how large -- maximizes aggregate throughput across every QP sharing the adapter, which is a reasonable design goal in isolation. The cost, measured directly by the finding above, is that "maximize aggregate throughput" and "protect any individual QP's throughput from its neighbors" are different goals that can actively conflict: the same work-conserving behavior that keeps the pipeline maximally busy is exactly what lets one large flow's WQEs monopolize a shared, finite resource at a small flow's direct expense. A design that instead reserved dedicated per-QP pipeline capacity would fix the isolation problem at the cost of the aggregate-throughput goal the work-conserving design optimized for -- capacity reserved for an idle QP is capacity nothing else can use in the meantime. Neither choice is free, and which one a real vendor's default behavior favors is exactly what the next chapter's scheduling-policy question is actually about.

[Multi-QP TX Scheduling](/tx/multi-qp-tx-scheduling) is next: the policy question this chapter's confirmed collapse directly motivates.

## References

- No IETF or IBTA standard governs internal TX-pipeline stall behavior or queue-slot allocation -- it is implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above, not a standards document.
