---
title: "TX Packet Build"
weight: 4
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "Every other chapter in this part is about getting a WQE on-chip. This one is about what happens the instant it is: the stage that actually turns a fetched descriptor into the BTH/RETH/AETH packet this book described field by field back in Part 0, and closes it out with an ICRC covering exactly the fields that survive the trip unchanged."
prev: /tx/wqe-prefetch
next: /tx/single-qp-pipeline-stalls
---

## Overview

This stage is the last step of a fetch-produce-issue sequence, run for RDMA specifically: everything the previous three chapters covered -- the WQE's fields, the ring holding it, the on-chip prefetch getting it there ahead of need -- exists to feed this one stage, turning a fetched WQE into the exact wire packet [What Is RoCEv2](/overview/what-is-rocev2) already described byte-for-byte. This is where a WQE's opcode and scatter-gather list actually become a Base Transport Header, a RETH if the opcode needs one, and a closing ICRC -- the concrete moment software's shape and hardware's wire shape finally converge.

> [!WARNING] Analogy
> A doorbell is the same kind of "kick" any command-queue-driven hardware engine responds to: fetch the descriptor, produce the fixed output format it requires, issue it.

## Locator

![The master RNIC block diagram with TX Packet Build highlighted -- this chapter is the one stage that actually produces the wire packet every other chapter in this book has described the shape of.](/images/tx/04-tx-packet-build-locator.svg)

[WQE Prefetch](/tx/wqe-prefetch) covered getting a WQE on-chip ahead of need. This chapter is what happens the instant it's needed: the stage that turns a fetched WQE into the wire packet [What Is RoCEv2](/overview/what-is-rocev2) already described field by field.

## Block diagram

![A fetched WQE fanning out into BTH (opcode, dest QP, and PSN pulled in from QP Context), a conditional RETH carrying the WQE's own remote address and rkey on FIRST/ONLY packets only, and a payload segment either inline or DMA-read -- all three converging into an assembled packet that gets an ICRC computed last over its invariant fields before leaving as wire output.](/images/tx/04-tx-packet-build-block.svg)

## Deep walkthrough

**What this stage actually assembles, field by field, already specified elsewhere.** A Base Transport Header carries this WQE's opcode, the destination-QP field this QP fixed once at INIT→RTR ([QP Context](/qp/qp-context)) and never revisits per-packet, and a PSN drawn from this QP's own live send-side PSN counter -- advancing under hardware's own control, no software write involved, exactly as [QP Context](/qp/qp-context) already establishes. If this WQE's opcode requires one -- an RDMA Write or Read, on the first or only packet of the message -- a RETH carries the remote virtual address and the `rkey` the WQE itself supplied. This side does not resolve that `rkey` against anything: it is an opaque value naming a Memory Region on the *remote* peer, forwarded as-is: the responder is who resolves it against its own Key Table on arrival, exactly the way [The Key Table](/mr/the-key-table) describes for the remote-side case -- the mirror image of the local `lkey` lookup this side performs for its own local reads and writes. None of these fields' shapes are new information this chapter introduces -- [What Is RoCEv2](/overview/what-is-rocev2) already fixed every bit width and byte offset, and [The Operation Catalog](/overview/operation-catalog) already fixed which opcodes carry which extended headers. This chapter's actual job is narrower and more honest about it: naming that a real pipeline stage has to *produce* those already-specified shapes from a WQE's fields, not re-deriving the shapes themselves.

**Where this is checked, and the honest limit of what's public here.** No source this book found describes the internal microarchitecture of this specific stage -- whether BTH, RETH, and payload assembly happen as one combined operation or several internal pipeline sub-stages, whether a single hardware unit builds every opcode's packet or different opcodes route through different logic, is not publicly documented for any real RNIC. This is the same honest limit [QP Table](/qp/the-qp-table) and [Key Table](/mr/the-key-table) already hit describing their own internal addressing mechanism -- a real design choice exists, and no vendor's public documentation commits to one. What can be said with confidence, because it follows directly from what every opcode's required fields already are: this stage cannot complete before the fields it needs -- this QP's live PSN, a resolved `rkey` if one's required, the payload itself -- are all actually available, which is exactly the dependency [Single-QP Pipeline Stalls](/tx/single-qp-pipeline-stalls) picks up next.

**ICRC -- computed last, and only over what actually survives the trip.** [What Is RoCEv2](/overview/what-is-rocev2) already established which fields the invariant CRC covers -- everything except the handful of fields a router or switch is allowed to modify in flight (Ethernet addresses, IP TTL/hop limit, and similar mutable framing). This stage is where that computation actually has to happen, and it has to happen *after* every covered field has its final value -- a BTH/RETH built with a placeholder PSN or an unresolved address, then patched afterward, would need the ICRC recomputed against the final bytes, not the placeholder ones. This chapter doesn't resolve whether a real pipeline computes ICRC incrementally as each covered field is finalized or in one pass at the end -- another instance of the same internal-sequencing question the paragraph above already flagged as undocumented.

**What happens next.** A built, ICRC-closed packet is what actually leaves the port -- the exact bytes [What Is RoCEv2](/overview/what-is-rocev2) described crossing the wire, now a real instance of that shape rather than a description of it. This chapter's job ends here; what happens if this stage never gets to run at all -- because the WQE feeding it stalled, or the QP feeding it reset mid-build -- belongs to the next two chapters, not this one.

> [!TIP] Why
> Fixing every wire-format field's shape and position at the specification level ([What Is RoCEv2](/overview/what-is-rocev2)), independent of how any given vendor's pipeline actually produces it, is what lets RoCEv2 interoperate across vendors at all -- two RNICs built by different companies, with completely different internal packet-build pipelines, only have to agree on the bytes that leave the port, not on how either of them assembles those bytes internally. That's exactly why this chapter can state every field's shape with total confidence while stating the assembly mechanism itself as undocumented -- the specification was only ever a contract about the wire, never a contract about the silicon producing it, and a vendor is free to build this stage as one monolithic unit or a dozen internal sub-stages as long as the packet that leaves matches the spec bit-for-bit.

[Single-QP Pipeline Stalls](/tx/single-qp-pipeline-stalls) is next: what happens when one of this stage's required inputs -- a resolved `rkey`, an available PSN, the payload itself -- isn't ready yet, and whether that stall touches any other QP sharing this same pipeline.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
- InfiniBand Trade Association, *Supplement to InfiniBand Architecture Specification Volume 1, Release 1.2.1, Annex A17: RoCEv2* (2014).
