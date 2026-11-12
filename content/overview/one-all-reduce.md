---
title: "One All-Reduce, Four Hosts"
weight: 6
date: 2026-11-11
publishDate: 2026-11-11
draft: false
description: "Distributed training spends much of its network time in one collective: every GPU ends up with the sum of every GPU's buffer. Here is that all-reduce done the common way, as a ring over four hosts and a switch, with each hop broken down into the verbs an RNIC actually executes."
prev: /overview/one-transaction-start-to-finish
next: /overview/how-to-read-this-guide
metaDescription: "A ring all-reduce over four hosts and a switch, broken into the RDMA operations NCCL issues per hop: a clear-to-send WRITE and a WRITE with Immediate."
seoTitle: "Ring All-Reduce over RDMA, Step by Step"
---

[One Transaction, Start to Finish](../one-transaction-start-to-finish) followed one operation between two hosts. Much of the traffic a modern RNIC carries is not shaped like that. A training job on many GPUs ends each step with an all-reduce: every GPU holds a buffer of gradients, and every GPU needs the element-wise sum of all of them. Nothing in an all-reduce is new hardware. It is ordinary RDMA WRITEs and receive completions, issued by a library in a fixed pattern across many Queue Pairs at once. This article walks one all-reduce over four hosts on a single switch and names the chapter that owns each piece.

## The ring

The simplest way to sum four buffers is to send them all to one host, add them there, and send the result back. That host's link then carries three buffers in and three out while every other link sits mostly idle. The ring algorithm spreads the work evenly. Each buffer is cut into as many chunks as there are ranks, and the ranks are arranged in a ring, each sending only to its right-hand neighbour.

The first half, *reduce-scatter*, takes N-1 steps. In each step every rank sends one chunk to its right and adds the chunk arriving from its left into its own copy. A different chunk travels from each rank, so after N-1 steps each rank holds one chunk that has had every rank's contribution added in. The second half, *all-gather*, takes N-1 more steps that pass those finished chunks around the ring until every rank has all of them.

Each rank sends 2(N-1) chunks of size 1/N of the buffer: 2(N-1)/N of the buffer in total. That is under twice the buffer however many ranks join, which makes the ring bandwidth-optimal for this problem ([Patarasuk and Yuan](#references)). The cost is latency: 2(N-1) steps that must happen in order, which is why libraries switch to tree-shaped algorithms for small buffers.

## What one hop asks of the NIC

NVIDIA's NCCL library is the common implementation, and its InfiniBand transport, which also drives RoCE NICs, turns every hop into the same two operations.

**The receiver says where to write.** Before any data moves, the receiving rank posts a Receive WQE on the connection, then RDMA WRITEs a small slot into a FIFO in the sender's memory. The slot holds the address of the buffer the data should land in, the rkey that buffer was registered with, the expected size and a tag. The slot is a few dozen bytes. By default the NIC gathers it from a registered copy in the receiver's memory; NCCL can instead copy it into the WQE inline ([The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor)), but only when asked to. Most of these writes are unsignaled: NCCL requests a completion on only one slot in each pass round the FIFO, often enough to keep the Send Queue from filling with work it can never retire.

**The sender writes the chunk with an immediate.** Once the sender sees a fresh slot, it RDMA WRITEs the chunk straight to that address with that rkey. The last work request of the transfer is an RDMA WRITE with Immediate, signaled, carrying the transfer's size or request ID as the immediate value. On the wire that is the [RDMA Write](/overview/operation-catalog/rdma-write) family: a 1 MiB chunk at a 4,096-byte path MTU becomes a WRITE First, 254 WRITE Middles and a WRITE Last with Immediate. At the receiver the payload scatters to the RETH-named address ([The Scatter Path](/dma/the-scatter-path)), and the immediate consumes the Receive WQE posted earlier and produces a receive CQE ([CQE Generation](/cq/cqe-generation)). That CQE is how the receiving side learns the chunk has landed.

**The GPU does the arithmetic.** The NIC never adds anything. The data lands in a staging buffer, and a GPU kernel adds it into the rank's own chunk during reduce-scatter, or copies it during all-gather. With GPUDirect RDMA that staging buffer sits in GPU memory and the NIC writes it across PCIe without touching host memory; otherwise it sits in host memory and the GPU reads it from there. Either way, as far as the NIC is concerned, an all-reduce is a stream of one-sided writes.

The walk below plays the whole collective. Rank r starts with 10 to the power r in every chunk, so a partial sum shows by its digits which ranks have been added in, and a finished chunk reads 1111.

{{< flow "allreduce" >}}

## What the switch sees

In every step of a single ring, each rank sends to exactly one neighbour and receives from exactly one. Every output port on the switch therefore has one sender aimed at it, and the many-to-one pile-up that [CNP Generation](/congestion/cnp-generation)'s incast walk builds does not form. All four links are busy at once, each in both directions: data flowing one way round the ring and clear-to-send slots flowing the other.

That clean picture holds for one ring on one switch. NCCL runs several rings, or channels, in parallel to use every link, can spread one connection over more than one Queue Pair, and a real cluster puts many jobs on a multi-tier fabric where hops between racks share uplinks. Collisions come back there, and so does the congestion control of [Congestion Control Engine](/congestion/). Some switches go further and perform the reduction themselves, so that each host sends its buffer once and receives the sum ([Graham et al.](#references)). That is a different division of labour between NIC and switch, and this article does not cover it.

## What the walk leaves out

NCCL cuts each chunk into smaller slices and pipelines them, so a rank is forwarding one slice while the next is still arriving; the walk sends each chunk whole. It also shows one channel and one Queue Pair per connection, leaves out the tree algorithms used for small buffers, and does not show how buffers are registered when the job starts. None of those change what the NIC executes per hop: a small WRITE in one direction, a large WRITE with Immediate in the other, and a receive completion.

## References

- NVIDIA, *NCCL* source, `src/transport/net_ib/p2p.cc`, `connect.cc` and `common.h`, commit `ad2d192` (November 2026) -- the receiver fills an `ncclIbSendFifo` slot (address, per-device rkeys, size, tag, index) and posts it with an `IBV_WR_RDMA_WRITE`, inline only when `NCCL_IB_USE_INLINE` is set (default off) and signaled on one slot per device; the sender posts the data as `IBV_WR_RDMA_WRITE` with the final work request `IBV_WR_RDMA_WRITE_WITH_IMM`, signaled, whose immediate carries the size or request ID; the receiver posts a receive work request per transfer, or preposts them, and matches the `IBV_WC_RECV_RDMA_WITH_IMM` completion. https://github.com/NVIDIA/nccl
- P. Patarasuk and X. Yuan, "Bandwidth optimal all-reduce algorithms for clusters of workstations," *Journal of Parallel and Distributed Computing*, 69(2), 117-124 (2009).
- R. L. Graham et al., "Scalable Hierarchical Aggregation Protocol (SHArP): A Hardware Architecture for Efficient Data Reduction," *First International Workshop on Communication Optimizations in HPC (COMHPC)*, 2016.
- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 -- RDMA WRITE with Immediate and its consumption of a Receive WQE.
