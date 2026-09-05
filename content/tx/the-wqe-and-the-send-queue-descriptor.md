---
title: "The WQE and the Send Queue Descriptor"
weight: 1
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "A WQE and a wire packet are not the same shape. Software builds one -- a generic, four-field struct any transport can use. Hardware builds the other from it -- opcode, BTH, RETH, payload, ICRC. This chapter is about the shape that exists in between the two: what a Send Queue actually holds before any of that translation happens."
next: /tx/the-send-queue-ring
---

## Locator

![The master RNIC block diagram with Host Memory / Descriptor Rings highlighted and every other block dimmed -- this chapter covers what one ring entry actually contains, not the ring structure itself.](/images/tx/01-the-wqe-and-the-send-queue-descriptor-locator.svg)

[One Transaction, Start to Finish](/overview/one-transaction-start-to-finish) already walked a Work Request from posting to wire, and named the doorbell that tells hardware new work exists without saying what, exactly, that work looks like sitting in memory. This chapter opens that box: the WQE itself, before any pipeline stage has touched it.

## Overview

A Work Queue Element is what software actually writes into a Send Queue ring -- a generic, transport-agnostic description of one operation: an opcode, a scatter-gather list naming source bytes, and whatever remote-addressing fields that opcode needs. It is not the wire packet [What Is RoCEv2](/overview/what-is-rocev2) already described field by field, and it is not necessarily what a specific RNIC's own silicon stores on-chip either -- three genuinely different shapes sit between "software posts a request" and "a BTH/RETH/AETH packet leaves the port," and this chapter is about the first of them.

## Block diagram

![Software's generic ibv_send_wr/ib_sge shape (opcode, sg_list/num_sge, remote addr/rkey, send_flags) translating into Mellanox mlx5's real four-field on-chip control segment (opcode, sq_ds, flags, misc, with a 6-bit data-segment count capping a WQE at 63 segments), which then forks on the 28-byte inline threshold into either an inline payload carried in the WQE itself or a separate DMA read.](/images/tx/01-the-wqe-and-the-send-queue-descriptor-block.svg)

## Deep walkthrough

**The software-facing shape, and what it deliberately doesn't commit to.** The Linux kernel's own generic representation, `struct ib_send_wr` (and its userspace-verbs mirror, `struct ibv_send_wr`), is deliberately thin: a `next` pointer for chaining multiple WRs in one post call, a `wr_id`/`wr_cqe` for matching a future completion back to this specific request, a `sg_list`/`num_sge` pair naming local source bytes via a `struct ib_sge` (`addr`, `length`, `lkey` -- the local key [The Memory Region Descriptor](/mr/the-memory-region-descriptor) already covers field by field), an `opcode` from the catalog [0.4](/overview/operation-catalog) already enumerated, `send_flags`, and a union carrying either immediate data or an invalidate-rkey value depending on which opcode this is. Nowhere in this struct is there a BTH, a PSN, or an ICRC -- those are hardware's job to produce, not software's job to supply. This is the same abstraction-boundary gap this book has already named at the doorbell ([0.5](/overview/one-transaction-start-to-finish)) and the QP table's internal organization ([2.2](/qp/the-qp-table)): a public, checkable software-facing struct exists, and it is not a claim about what any specific RNIC stores internally.

**A real vendor's on-chip shape, shown specifically to make that gap concrete rather than asserted abstractly.** Mellanox's mlx5 driver packs a WQE's control segment (`mlx5_wqe_cseg`, confirmed directly against the DPDK source) into exactly four 32-bit fields: `opcode`, `sq_ds`, `flags`, `misc`. `sq_ds` folds two pieces of information into one field -- how many data segments this WQE carries, constrained to 6 bits, capping a single WQE at 63 data segments (`MLX5_DSEG_MAX`). Compare this to `ib_send_wr`'s separate, independent `num_sge` field: the same information (how many scatter-gather entries this operation touches) exists in both shapes, but one vendor's real hardware packs it into a shared, width-constrained field alongside an opcode, while the software-facing struct keeps it as its own untyped `int`. Neither shape is wrong -- they're solving different problems. Software's shape optimizes for a stable, transport-agnostic API surface across every vendor's hardware; one real vendor's on-chip shape optimizes for what actually fits efficiently into a fixed-width control segment a pipeline stage can parse in hardware. This is one real, checkable example, not a claim that every RNIC packs a WQE control segment identically.

**Measured sizes -- concrete numbers, not qualitative gestures.** Kong et al. (NSDI 2023), measuring real ConnectX hardware directly, give this chapter its interface-table anchor: "A WQE for RC/UC is 36-byte, and 68-byte for UD." UD's larger size tracks directly with what [QP Context](/qp/qp-context) already established -- a UD QP names its destination per-message rather than once at connection setup, so its WQE has to carry addressing information RC/UC's WQE doesn't need, because RC/UC already fixed the destination QP number at INIT→RTR.

**The inline/DMA fork -- a real threshold with a real hardware consequence.** The same paper measures a "maximal inline size (28-byte)": a WQE whose payload fits within that threshold carries the payload bytes directly inside the WQE itself, avoiding a separate PCIe transaction to fetch it; a payload above that threshold requires the adapter to issue its own DMA read for the data, a mechanism [The Send Queue Ring](/tx/the-send-queue-ring) picks up from the fetch side and Part 8's DMA Engine owns in full. This is the first of several places in this Part where a small, fixed-size number determines whether an operation costs one memory transaction or two -- worth carrying forward precisely rather than treating "the adapter fetches the WQE" as the end of the story.

**Where this is checked, and what happens next.** Nothing about the WQE's own fields is validated in isolation -- the opcode has to be one this QP's transport actually supports (Section [0.4](/overview/operation-catalog)'s catalog, gated by the QP's own transport type from [QP Context](/qp/qp-context)), the `lkey` on every SGE has to resolve against a real, permission-matching Memory Region entry ([The Key Table](/mr/the-key-table)), and the QP itself has to be in RTS ([QP State Machine](/qp/qp-state-machine)) before any of this is allowed to leave host memory at all. This chapter's job stops at naming what the WQE contains and how its shape differs from both the software struct above it and the wire packet below it; [The Send Queue Ring](/tx/the-send-queue-ring) is where a WQE actually gets fetched off the structure holding it.

> **Design rationale.** Keeping the software-facing WQE struct thin and transport-agnostic, rather than exposing a variant shaped to each vendor's actual on-chip packing, is what lets one verbs API target every RNIC vendor without an application recompiling against vendor-specific structs. The cost of that abstraction is paid entirely on the hardware side of the boundary: every real implementation has to translate a generic `opcode` + `sg_list` shape into whatever dense, fixed-width packing its own pipeline can parse efficiently -- mlx5's four-32-bit-field control segment is one real, public example of paying that cost, not the only way to pay it. A vendor that instead exposed its literal on-chip WQE format as the API would save that translation step, at the cost of every application becoming vendor-locked to that one layout -- exactly the tradeoff RDMA's whole verbs model was built to avoid.

## Interface reference

| Field | Source | Notes |
|---|---|---|
| `opcode` | `struct ib_send_wr` | from the catalog ([0.4](/overview/operation-catalog)); must match this QP's transport |
| `sg_list` / `num_sge` | `struct ib_send_wr` | local scatter-gather list, each entry an `addr`/`length`/`lkey` triple |
| `send_flags` | `struct ib_send_wr` | e.g. signaled-completion request |
| immediate data / invalidate rkey | `struct ib_send_wr` (union) | opcode-dependent, mutually exclusive |
| mlx5 control segment | `mlx5_wqe_cseg` (DPDK `mlx5_prm.h`) | 4x 32-bit: `opcode`, `sq_ds`, `flags`, `misc` -- one real vendor's on-chip packing |
| Data segment (DS) count width | `mlx5_wqe_cseg.sq_ds` | 6 bits, `MLX5_DSEG_MAX` = 63 segments per WQE |
| RC/UC WQE size | measured (Kong et al., NSDI 2023) | 36 bytes |
| UD WQE size | measured (Kong et al., NSDI 2023) | 68 bytes -- larger due to per-message destination addressing |
| Maximum inline payload | measured (Kong et al., NSDI 2023) | 28 bytes -- above this, payload requires a separate DMA read |

[The Send Queue Ring](/tx/the-send-queue-ring) is next: the structure that holds one of these WQEs per posted entry, and how hardware finds the next one to fetch.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
