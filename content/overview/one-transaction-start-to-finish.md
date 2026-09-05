---
title: "One Transaction, Start to Finish"
weight: 5
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "Every mechanism in this book gets its own chapter, in isolation. This is the one place they run together — one RDMA Write, start to finish, with a map of exactly which later chapter owns each stage it passes through."
prev: /overview/operation-catalog
next: /overview/how-to-read-this-guide
---

The last four articles built the pieces separately: what RDMA demands of hardware that a conventional NIC never had to provide ([0.1](../what-is-rdma)), the exact byte layout of a RoCEv2 packet ([0.2](../what-is-rocev2)), why that layout carries an IP/UDP header at all ([0.3](../rocev1-to-rocev2)), and the full catalog of operations a Queue Pair can issue ([0.4](../operation-catalog)). None of them showed one operation actually moving through hardware, front to back. This article does exactly that — one RDMA Write with Immediate, from an application posting a Work Queue Element to the completion landing back in that same application's Completion Queue — and names, at every stage, which later Part of this book owns that stage in depth. Nothing here is explained fully; that's deliberate. This article is a map, not a chapter.

## Bring-up: everything that has to exist first

Before any operation can move, four objects have to exist and one handshake has to complete. The application registers a Memory Region against the buffer it intends to write into — pinning the pages and handing the adapter a virtual-to-physical mapping along with an `lkey`/`rkey` pair encoding what's allowed against that memory (owned by [Part 2](/mr/), chapter 4, *Registering a Memory Region*). It creates a Completion Queue to receive completion notifications (Part 7 (Completion Queue / Event Management)). It creates a Queue Pair and drives it through the hardware-enforced sequence RESET → INIT → RTR → RTS, supplying at each transition the specific attributes that transition requires — destination QP number and path MTU at INIT→RTR, retry count and ACK timeout at RTR→RTS ([Part 1](/qp/) (Queue Pair Management), chapters 4 and 5, *Installing a QP* and *QP State Machine*). Only once the QP reports RTS can it send anything at all. Everything that follows assumes this has already happened on both ends of the connection.

## The send path: a WQE becomes a packet

The application posts a Work Request describing an RDMA Write with Immediate: a local scatter-gather entry naming the source bytes and their `lkey`, a remote address and `rkey` naming where those bytes should land, and a 32-bit immediate value the receiver will see without having to inspect the payload. Posting doesn't move data — it writes a descriptor into a ring in host memory and rings a doorbell, an architectural producer-index write telling the adapter that new work exists (the exact register-level mechanism is vendor-specific and not part of any public interface; this book describes it functionally). Fetching that descriptor off the ring and managing the ring itself is Part 3's opening concern (*TX Pipeline*).

Before the adapter can build a packet, it needs the sending QP's live context — its current PSN, its negotiated path, its state — and getting that context on-chip cheaply, or paying for a miss when it isn't already cached, is the whole subject of Part 1, chapters 2 and 3 (*The QP Table*, *QP Caching*). The adapter also has to validate the local scatter-gather entry against its `lkey` before it's allowed to read the source bytes at all — the same Key Table that Part 2 spends its first three chapters on, consulted here from the sending side rather than the receiving side.

With permission and context both resolved, the adapter builds the wire packet: a Base Transport Header carrying the opcode, the destination QP, and the PSN drawn from the QP's live send-sequence state; an RDMA Extended Transport Header carrying the remote address, `rkey`, and length (present here because this message fits in one packet — a longer message would carry RETH only on its first packet, per [0.4](../operation-catalog)); the immediate value; and the payload itself, closed out with the invariant CRC described in [0.2](../what-is-rocev2). Before that packet is allowed to leave, Part 6's rate limiter (*Congestion Control Engine*) may hold or pace it against that QP's current congestion state — a gate this walkthrough passes through without stopping to examine, since Part 6 owns what that gate is actually doing. Because this is a Reliable Connection operation, the adapter also can't simply forget the packet once it's sent: a copy has to be held somewhere until it's acknowledged, and a retry timer has to start. That holding structure — the replay buffer — and the timer it's paired with belong entirely to Part 5 (*Reliability Engine*).

## The wire

What actually goes out the port — Ethernet, then IP, then UDP to the fixed RoCEv2 destination port, then the InfiniBand transport headers just assembled, then payload, then ICRC — is exactly the packet [0.2](../what-is-rocev2) already described field by field. This article doesn't re-derive it; it just confirms that the packet built above is a real instance of that shape, now in flight.

## The receive path: a packet becomes bytes in memory

On the far side, the RX pipeline (Part 4 (RX Pipeline)) parses the Ethernet, IP, UDP, and BTH layers off the wire and reads the destination QP number out of the BTH — which drives the same kind of QP Table/Cache lookup Part 1 covers, now performed on the receiving QP instead of the sending one. Part 4 also owns a decision this walkthrough deliberately doesn't resolve: exactly where in the pipeline the incoming PSN gets compared against that QP's next-expected PSN, and what happens to a packet that arrives out of order. Assume, for this walkthrough, that the packet arrives in order and the check passes.

With the packet accepted, its RETH is resolved against the receiver's Key Table: the `rkey` has to map to a real, valid Memory Region, the requested write has to fall within that region's registered bounds, and the region's permission bits have to allow a remote write at all — three separate ways this step can fail, each owned by Part 2, chapter 5 (*Permission and Bounds Violations*). Only once all three pass does Part 8's DMA engine (*DMA Engine*) translate the target virtual address and actually place the payload in host memory, walking a scatter-gather descriptor chain if the destination region isn't backed by physically contiguous pages.

Because this operation carries an immediate value, it does something a plain RDMA Write does not: it consumes a Receive Request the application posted in advance on its receive queue, and it generates a receive-side completion carrying that immediate value — the application learns data arrived, and what the sender wanted it to know about that arrival, without having posted any work request that names the write itself. Generating that completion, and everything about how a Completion Queue holds and reports it, is Part 7's entire subject.

## The acknowledgment closes the loop

A Reliable Connection can't leave the sender in the dark. The receiver owes an acknowledgment, a Base Transport Header plus an ACK Extended Transport Header carrying a syndrome byte and a message sequence number, built and sent back the same way any other packet is — through the same TX path this walkthrough already crossed once, just carrying no payload. When that ACK reaches the original sender, it's the reliability engine (Part 5 (Reliability Engine), again) that matches it against the outstanding replay-buffer entry by PSN, retires that entry, and disarms the retry timer that was armed when the packet first went out. Only then, if the application asked to be signaled for this specific work request, does a send-side completion land in the sender's own Completion Queue (Part 7, once more) — the last event in the sequence, and the one the application was actually waiting to observe.

## What just happened, seen from above

One operation, and the QP Table/Cache pattern from Part 1 got consulted twice — once on each end. The Key Table from Part 2 got consulted twice as well, once to validate a local read and once to validate a remote write, by two different pieces of hardware on two different machines. The replay buffer from Part 5 held state for the entire round trip and nothing else in this walkthrough held state anywhere near that long. That's not a coincidence specific to RDMA Write — it's the same storage-structure-plus-working-set shape showing up in more than one place in this single transaction, and it shows up again, differently dressed, in several of the Parts this walkthrough only named in passing. The next article names that shape directly, so the rest of this book reads as a pattern applied repeatedly rather than eight unrelated subsystems that happen to sit next to each other on the same die.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
- InfiniBand Trade Association, *Supplement to InfiniBand Architecture Specification Volume 1, Release 1.2.1, Annex A17: RoCEv2* (2014).
- J. Postel, *Internet Protocol*, RFC 791, IETF (September 1981).
- J. Postel, *User Datagram Protocol*, RFC 768, IETF (August 1980).
- Internet Assigned Numbers Authority, *Service Name and Transport Protocol Port Number Registry* (UDP port 4791, "roce").
