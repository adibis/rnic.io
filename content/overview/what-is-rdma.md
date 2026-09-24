---
title: "What Is RDMA"
weight: 1
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "A normal NIC hands a frame to software and stops there. An RDMA-capable NIC is handed a virtual address and a remote key and is expected to move bytes into another machine's memory without a syscall, without the remote CPU, and without a single unverified byte -- this book is about what building that actually takes in hardware."
next: /overview/what-is-rocev2
---

A conventional NIC moves frames between a wire and a ring buffer and stops there — every byte that means something is somebody else's problem, decided in software after the frame has already landed. An RDMA-capable NIC cannot stop there. It is handed a virtual address, a length, and a remote key, and it is expected to move that many bytes into or out of a specific region of another machine's memory without a syscall on the data path, without the remote CPU noticing, and without ever moving a byte the key didn't authorize. That is not a faster NIC. It is a NIC doing work that used to belong entirely to an operating system's network stack, done instead in hardware — and that shift, block by block, is what this book is about. Most RDMA guides describe that shift as "the hardware handles reliability, address translation, and delivery" and consider the matter settled. This one opens each of those verbs into the pipeline stage, cache, and state machine actually doing the work, one part at a time. This chapter names the four capabilities that shift requires before any of the later parts start building them.

![Four rows comparing a conventional NIC against an RDMA-capable one: getting onto the data path (syscall vs. a directly-posted Work Request), checking memory access is allowed (the OS's page tables vs. the adapter's own key check), making delivery reliable (a software TCP stack vs. the adapter's own retry engine), and the remote side's involvement (both CPUs must participate vs. the remote CPU can be asleep).](/images/overview/01-what-is-rdma-compare.svg)

## Kernel bypass moves the data path out of the OS

A conventional socket write is a syscall: the application hands the kernel a buffer, the kernel copies it, queues it, and a driver eventually tells the NIC. Every one of those steps costs a context switch. RDMA's programming model removes the kernel from that path entirely — an application posts a Work Request directly onto a Queue Pair and polls a Completion Queue directly for the result, both ordinary local operations against memory the application already has mapped, not syscalls. The kernel is still involved in setting the path up — allocating the Queue Pair, registering memory — but once a QP is in a state where it can carry traffic, the operating system is no longer on the critical path of a single operation. Everything downstream of that fact is a hardware problem: something has to accept a Work Request posted directly by userspace, without kernel mediation, and act on it correctly. [Queue Pair Management](/qp/) and [TX Pipeline](/tx/) are where that something lives.

## Memory protection moves from the OS into the adapter

Removing the kernel from the data path removes the thing that used to check every access against a process's page tables. RDMA replaces that check with one done once, up front, instead of once per operation: memory registration. An application registers a region of its virtual address space with the adapter, the adapter pins the underlying pages and records a virtual-to-physical mapping, and registration returns a pair of keys — an `lkey` for the local side to reference that region in its own Work Requests, and an `rkey` a remote peer can present to reach the same region directly. Every operation against that memory afterward is checked against the key it carries, in hardware, not re-validated by an OS that is no longer in the loop. That single design decision is the reason [Memory Region / Key Table](/mr/) exists as its own part: a table the adapter must maintain, cache, and enforce permissions against.

> [!WARNING] Analogy
> This table does functionally the same job an MMU does for a CPU — just triggered by a key presented over the wire instead of a page-table walk issued by a local instruction.

## Reliable delivery moves from a software stack into the adapter's own retry engine

For a Reliable Connection (RC) Queue Pair, the adapter tracks its own packet sequence numbers, generates and consumes its own acknowledgements, and retries on its own — the job a software TCP stack normally does, run instead as a hardware state machine per connection. This isn't inference from the API shape alone: the wire header ([What Is RoCEv2](../what-is-rocev2)) carries a PSN field on every packet, and a QP's own configuration (`timeout`, `retry_cnt`, `rnr_retry` — [QP Context](/qp/qp-context)) only makes sense if something in the adapter is actively running a timer and a retry count against them. [Reliability Engine](/reliability/) is that state machine, and [Queue Pair Management](/qp/)'s [Single-QP Error Handling](/qp/single-qp-error-handling) and [Multi-QP Error Arbitration](/qp/multi-qp-error-arbitration) chapters cover what happens when it gives up.

## The remote CPU is optional

Two-sided operations — SEND and RECEIVE — still involve both ends' software: the sender posts a Send Request, the receiver has to have already posted a matching Receive Request for the payload to land in. RDMA WRITE and RDMA READ don't work that way. A WRITE names a remote virtual address and `rkey` directly, and the responding adapter performs the memory access itself, against a region registered in advance, without the responder's software posting a Work Request for that specific operation — a plain RDMA WRITE consumes no Receive Request at all. The remote CPU can be asleep, busy, or entirely uninvolved, and the data still arrives. That's the capability the "R" in RDMA is actually naming, and it's also why memory registration (previous section) has to happen *before* the operation, not during it — there's no remote software in the loop at operation time to authorize anything on the fly.

> [!NOTE] Concept
> A plain RDMA WRITE consumes no Receive Request at all. The remote CPU can be asleep, busy, or entirely uninvolved, and the data still arrives — that's the capability the "R" in RDMA is actually naming.

Four capabilities, one underlying fact: work that used to happen in an operating system, once per operation, on both ends of a connection, now has to happen in hardware, on every packet, with no software checkpoint to fall back on if something goes wrong. [What Is RoCEv2](../what-is-rocev2) puts a concrete wire format underneath all four — the header fields, byte offsets, and packet types that carry a Work Request from one adapter's SQ to another's RQ and back again as an acknowledgement.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
