---
title: "The Memory Region Descriptor"
weight: 1
date: 2026-09-04
publishDate: 2026-09-04
draft: true
description: "A registered memory region hands back two keys, not one -- and they are not two views of the same number. One unlocks local Work Requests against this process's own memory; the other is what a remote peer presents to touch it. This chapter is what's actually in the entry both of them point at."
next: /mr/the-key-table
---

[How to Read This Guide](/overview/how-to-read-this-guide) named the checklist [Part 1](/qp/) (Queue Pair Management) ran in full against the Queue Pair. This Part runs the same checklist a second time, against a different resource: the Memory Region and the key table that stores it. This chapter is where that starts -- the descriptor itself, field by field, before the next chapter covers the structure holding every one of them.

## Overview

A Memory Region descriptor is what a successful registration produces: a record of one virtual address range, the physical pages backing it, and exactly which operations -- local, remote, read, write, atomic -- are allowed against it. Software gets back two keys pointing at that one record, not one, and getting precise about what those two keys actually are, and are not, is this chapter's whole job.

## Locator

![The master RNIC block diagram with the MR/Key Table highlighted and Key Cache dimmed -- this chapter covers the data living inside one entry of that table, not the structure itself.](/images/mr/01-the-memory-region-descriptor-locator.svg)

Every field below lives inside one entry of the Key Table the next chapter covers -- this chapter is about what's in that entry, not the structure holding it.

## Block diagram

<!-- IMAGE: A single entry box labeled "MPT entry" subdivided into labeled sub-fields: "addr" (base virtual address), "length", "lkey", "rkey", "access flags" (shown as a small row of individually-labeled bit boxes: LOCAL_WRITE, REMOTE_WRITE, REMOTE_READ, REMOTE_ATOMIC, MW_BIND), "pd" (small icon linking out to a separate "Protection Domain" box). Two labeled arrows point INTO the lkey and rkey sub-fields specifically, from two separate actor boxes on the left drawn in the sky-blue actor color per docs/SVG_STYLE.md: "Local process (posts WRs against its own memory)" arrowing to lkey, "Remote peer (incoming RDMA operations)" arrowing to rkey -- both arrows terminate at the same entry box, visually establishing "one entry, two independently-scoped keys, two different callers." A separate small "handle" sub-field is shown off to one side in a dimmed/greyed box, captioned "software/kernel-side identifier for the MR object -- not a hardware-checked key, easy to conflate with lkey/rkey, isn't one." -->

## Deep walkthrough

**The full descriptor, and the field that's easy to conflate with the two that matter.** `struct ibv_mr`, the verbs-layer mirror of a registered Memory Region, carries: `context` and `pd` (which device and Protection Domain this MR belongs to), `addr` and `length` (the registered virtual range), `handle` (a software/kernel-side identifier for the MR object itself), and the two hardware-visible keys, `lkey` and `rkey`. `handle` is worth naming precisely so it doesn't get folded into "the key" by accident later: it identifies the MR object to software and the kernel, but it isn't presented on the wire, isn't checked by hardware against an incoming operation, and isn't one of the two things this chapter is actually about.

**`lkey` and `rkey` -- two keys, one entry, and they are not required to match.** Every successful registration returns a unique `lkey` and a unique `rkey`, independently assigned, not two views of the same underlying number. The clean way to hold this: one MR descriptor, one entry in the table the next chapter covers, and two differently-scoped keys pointing at it -- `lkey` for the local process to reference its own memory in a Work Request it posts itself, `rkey` for a remote peer to present in an incoming RDMA operation against that same memory. Same entry, different callers, independently issued keys, not a matched pair that happens to carry two names. (RDMA's iWARP lineage calls this same concept a "steering tag," or STag, a single term covering what RoCE/IB splits into the local/remote `lkey`/`rkey` pair -- worth knowing if a vendor doc or paper uses that word instead.) One driver-specific wrinkle worth flagging rather than generalizing from: the Linux kernel's software RoCE implementation (`rxe`) happens to set `lkey` equal to `rkey` internally, as a shortcut specific to its own soft-RDMA datapath. That's one driver's internal implementation choice, not something this book treats as representative of real RNIC hardware -- assume the two keys are independent unless a specific device's documentation says otherwise.

**Access flags -- what the entry actually allows, checked at the point of use.**
- *What*: a bitmask set at registration time -- `IBV_ACCESS_LOCAL_WRITE`, `IBV_ACCESS_REMOTE_WRITE`, `IBV_ACCESS_REMOTE_READ`, `IBV_ACCESS_REMOTE_ATOMIC`, `IBV_ACCESS_MW_BIND` (permission to bind a Memory Window against this MR), plus a handful of narrower-purpose flags this book doesn't dwell on (`IBV_ACCESS_ZERO_BASED`, `IBV_ACCESS_HUGETLB`, `IBV_ACCESS_RELAXED_ORDERING`). `IBV_ACCESS_ON_DEMAND` gets its own real treatment in [Registering a Memory Region](/mr/registering-a-memory-region) rather than a mention here. Newer flush-related flags (`IBV_ACCESS_FLUSH_GLOBAL`/`IBV_ACCESS_FLUSH_PERSISTENT`, oriented at persistent-memory access) are scoped out of this book's first pass, the same deliberate way [0.4](/overview/operation-catalog) scoped `ATOMIC_WRITE`/`FLUSH` out of the opcode catalog.
- *Where checked*: against the flags stamped into this same entry, whenever an operation presents this MR's `lkey` (locally) or `rkey` (remotely) and asks to do something with it.
- *When it fails*: an operation asking for something the flags don't allow -- a remote write against an MR registered without `IBV_ACCESS_REMOTE_WRITE`, say -- is a permission violation. [Permission and Bounds Violations](/mr/permission-and-bounds-violations) is the chapter that owns everything downstream of that, in depth.
- *What happens next*: this chapter's job stops at naming what's in the entry and what it permits; whether a specific access attempt against it is honored is the next few chapters' subject, not this one's.

## Design rationale

> **Design rationale.** Issuing `lkey` and `rkey` as two independent values, rather than one shared key checked differently depending on who presents it, is what lets the two permission surfaces move independently. A local process's own access to memory it registered itself is a much less sensitive question than what a remote peer -- untrusted, on a different machine, authenticated by nothing stronger than possession of a 32-bit value -- is allowed to do to that same memory. Splitting the key means revoking or never granting the remote-facing one (`rkey`) doesn't have to touch the local-facing one at all, and an application that only ever needs local access never has to hand out anything a remote peer could present. A single shared key would work too, checked against a caller-identity bit the hardware would then have to carry alongside every operation -- independent keys sidestep needing that extra piece of state at all, at the cost of managing two numbers instead of one.

## Interface reference

| Field | Source | Scope | Notes |
|---|---|---|---|
| `context` / `pd` | `struct ibv_mr` | software | which device and Protection Domain this MR belongs to |
| `addr` / `length` | `struct ibv_mr` | software + hardware | the registered virtual address range |
| `handle` | `struct ibv_mr` | software/kernel only | MR object identifier -- not wire-visible, not a hardware-checked key |
| `lkey` | `struct ibv_mr` | local process | presented in this process's own Work Requests against this MR |
| `rkey` | `struct ibv_mr` | remote peer | presented in an incoming RDMA operation against this MR |
| `IBV_ACCESS_LOCAL_WRITE` | access flags | local | allows local write via `lkey` |
| `IBV_ACCESS_REMOTE_WRITE` | access flags | remote | allows incoming RDMA Write via `rkey` |
| `IBV_ACCESS_REMOTE_READ` | access flags | remote | allows incoming RDMA Read via `rkey` |
| `IBV_ACCESS_REMOTE_ATOMIC` | access flags | remote | allows incoming Atomics via `rkey` |
| `IBV_ACCESS_MW_BIND` | access flags | local | allows a Memory Window to be bound against this MR |
| `IBV_ACCESS_ON_DEMAND` | access flags | local | ODP -- see [Registering a Memory Region](/mr/registering-a-memory-region) |

[The Key Table](/mr/the-key-table) is next: the structure that holds one of these entries per registered region, and how a presented `lkey` or `rkey` finds the right one.

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1 (General Specifications).
