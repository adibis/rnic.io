---
title: "Target Selection"
weight: 2
date: 2026-09-16
publishDate: 2026-09-16
draft: false
description: "A normal NIC's DMA engine only ever writes into a buffer software chose ahead of time. This engine sometimes does that too -- and sometimes places data at an address the wire packet itself names, resolved through a key software never touched at the moment the transfer runs. The opcode decides which. This is the one demand ordinary DMA hardware never had to meet."
prev: /dma/in-flight-transfer-state
next: /dma/address-translation-handoff
---

## Overview

Before this engine can move a single byte, it has to answer a question a normal NIC's DMA engine never has to ask: where does this transfer's address actually come from? Two answers exist. One is *software-named* -- the address sits in a scatter-gather entry a Work Queue Element already carries, chosen by software before the operation was ever posted. The other is *wire-named* -- the address arrives in the packet itself, in RETH's virtual address field, resolved against a key the receiving side had no hand in choosing at the moment the packet landed. Which one applies depends entirely on the opcode, and on which side of the connection is asking.

> [!NOTE] Concept
> Software-named addressing is what every DMA-capable device already does: a descriptor, written by software ahead of time, tells the engine exactly where to read from or write to. Wire-named addressing is what makes this engine specifically an RDMA engine -- the target of a memory access is decided by a packet that arrived over the network, at the instant it arrives, not by anything the receiving side's own software prepared in advance.

## Locator

![The master RNIC block diagram with DMA Engine highlighted and every other block dimmed -- this chapter covers how the engine decides which address a transfer actually resolves against, not the translation lookup that follows once it has.](/images/dma/02-target-selection-locator.svg)

Target selection happens before [Address Translation Handoff](/dma/address-translation-handoff), immediately next -- this chapter names which raw address enters that lookup, not what the lookup does with it.

## Block diagram

![A decision point keyed on opcode and direction: SEND source, WRITE source, RECEIVE destination, a READ response's destination at the requester, and an Atomic's return-value destination all resolve against a WQE's own SGE list or buffer (software-named, lkey); a WRITE's destination and a READ request's source at the responder both resolve against RETH's virtual address (wire-named, rkey) -- drawn as two converging paths into one shared translation stage.](/images/dma/02-target-selection-block.svg)

## Deep walkthrough

**The trap this chapter exists to correct.** It's tempting to assume gather (reading local memory to build an outgoing payload) is always software-named and scatter (writing incoming data somewhere) is always wire-named -- reading is something you do to your own stuff, writing is something the network tells you to do. That assumption is wrong for exactly one case, and it's an important one: an RDMA READ response's payload, gathered at the *responder*, is a read -- but the address it reads from came off the wire, in the READ request's own RETH, not from anything the responder's software posted. Direction and addressing source are independent questions; this chapter answers both separately, opcode by opcode.

> [!CAUTION] Gotcha
> Don't collapse "gather" and "software-named" into the same idea, or "scatter" and "wire-named" into the same idea. They correlate for SEND, RECEIVE, and a WRITE's two ends -- but a READ request's responder-side gather is wire-named, and that's precisely the case most likely to get waved through as "the same as WRITE, just reversed."

**SEND.** The source, at the sender, is software-named: a scatter-gather entry in the sending WQE, each segment an address, a length, and the local key [The Memory Region Descriptor](/mr/the-memory-region-descriptor) already covers field by field. The destination, at the receiver, is *also* software-named, just by a different side's software -- the receiving QP's already-posted Receive Queue WQE names where the incoming payload lands, resolved through that WQE's own SGE list and its own local key. No RETH exists on a SEND packet at all; the wire carries no addressing information for this opcode in either direction.

**RDMA WRITE (first/only).** The source, at the sender, is software-named, identically to SEND's source. The destination, at the receiver, is wire-named: RETH's virtual address and `rkey`, carried in the packet itself, resolved against whatever Memory Region that key names -- the receiver's own Receive Queue is never consulted, and (barring the immediate-data variant below) no Receive Queue Element is even consumed. This is the one opcode this book's own framing has pointed at from the start: a destination the receiving side's software never named at the moment the packet arrives.

**RDMA WRITE with Immediate.** Payload placement is identical to a plain WRITE -- wire-named, via RETH. The immediate value alone is what reaches software through the ordinary Receive Queue path: this variant *does* consume one posted Receive Queue Element, not to place the payload, but to give the immediate value and the resulting receive completion somewhere to attach to. Two independent addressing questions, answered two different ways, in the same packet.

**RDMA READ, request phase.** The requester posts a WQE naming two things at once: a local scatter-gather list (software-named, where the eventual response payload should land) and RETH's remote address and `rkey` (wire-named from the responder's perspective, naming what to read). At the responder, servicing the request is a gather -- but its source is wire-named: the address came from the request packet's own RETH, not from anything the responder's software posted.

**RDMA READ, response phase.** The requester's own DMA engine places the arriving response payload into the local buffers its original request WQE already named -- software-named, resolved back to the same scatter-gather list the request carried, held live for the round trip the same way [In-Flight Transfer State](/dma/in-flight-transfer-state)'s completion tag keeps an operation's identity alive across its own execution.

**Atomics.** The responder's side is wire-named, the same shape as a READ request: a single eight-byte remote address and `rkey`, carried in the packet, naming the one memory location the operation reads, modifies, and writes back. No scatter-gather list applies -- atomics have exactly one target, not a chain of segments. The pre-operation value returned to the requester lands in a local buffer the original WQE named, software-named, the same shape as a READ response.

> [!TIP] Why
> Letting a packet name its own target, instead of requiring the receiving side to have already posted a matching buffer, is exactly what makes RDMA WRITE and RDMA READ useful for the workloads that motivate this book in the first place: a remote peer can place or retrieve data at a location the local side registered once, without that side's software being anywhere in the loop for each individual transfer. SEND/RECEIVE's software-named-on-both-ends shape is the one this book's [Operation Catalog](/overview/operation-catalog) already covers as the closest RDMA gets to ordinary networking -- both sides' software stays in the loop, the same way it would for any other kind of message passing.

**Where this connects.** Every address this chapter resolves, wire-named or software-named, still has to become a physical address before a single byte moves -- that lookup, and the cache backing it, is [Address Translation Handoff](/dma/address-translation-handoff), immediately next. Which local key or which remote key actually gets checked is what [The Memory Region Descriptor](/mr/the-memory-region-descriptor) and [Permission and Bounds Violations](/mr/permission-and-bounds-violations) already cover in full -- this chapter only decides which address and which key enter that process, not what happens once they do.

## Interface reference

| Opcode | Source (gather) | Destination (scatter) |
|---|---|---|
| **SEND** | software-named -- sender's WQE, `lkey` | software-named -- receiver's posted RQ WQE, `lkey` |
| **RDMA WRITE** | software-named -- sender's WQE, `lkey` | wire-named -- RETH virtual address, `rkey` |
| **RDMA WRITE w/ Immediate** | software-named -- sender's WQE, `lkey` | wire-named for payload (RETH); receiver's posted RQ WQE consumed only for the immediate value |
| **RDMA READ (request)** | wire-named at responder -- RETH virtual address, `rkey` | software-named at requester -- original WQE's SGE list, `lkey`, resolved on response arrival |
| **Atomic** | wire-named at responder -- one 8-byte remote address, `rkey` | software-named at requester -- original WQE's buffer, `lkey` |

## References

- No IBTA or IETF standard names "target selection" as its own mechanism -- the opcode-by-opcode addressing behavior above is derived directly from RETH and ATMETH's own defined fields, already covered field by field in [The Operation Catalog](/overview/operation-catalog) and [One Transaction, Start to Finish](/overview/one-transaction-start-to-finish), applied here to the specific question of which side's data each opcode actually touches.
