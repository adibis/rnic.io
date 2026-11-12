---
title: "The Key Table"
weight: 2
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "A QP number is a dense index handed out by the adapter itself. A key is neither -- it's an opaque 32-bit value a remote peer presents from across the wire, and the table it has to resolve against is built to fail cleanly on a wrong one rather than fail dangerously into someone else's memory."
prev: /mr/the-memory-region-descriptor
next: /mr/key-caching
metaDescription: "The RNIC Key Table: how an opaque 32-bit lkey or rkey resolves to a Memory Region, built to fail cleanly on a wrong key rather than expose memory."
seoTitle: "The RDMA Key Table: lkey and rkey"
---

[The Memory Region Descriptor](/mr/the-memory-region-descriptor) covered what's inside one entry. This chapter is about the structure holding every entry, and how a presented key finds the right one.

## Overview

Every registered Memory Region's descriptor lives in a table sized to hold as many registrations as the adapter supports, resolved by whichever key -- `lkey` locally, `rkey` remotely -- an operation presents. Unlike [The QP Table](/qp/the-qp-table)'s QP number, a key isn't handed out by the adapter from a dense range it controls; it's a value the adapter itself assigns at registration and a remote peer simply presents back, unchanged, however long after that as its own hardware chooses to use it. That difference in where the value comes from and how long it can sit unused before being presented again is what makes this table's failure shape different from the QP Table's.

## Locator

![The master RNIC block diagram with the MR/Key Table highlighted and Key Cache dimmed -- this chapter is the backing store every Memory Region Descriptor lives in as one addressable entry.](/images/mr/02-the-key-table-locator.svg)

## Block diagram

![The MPT (Memory Protection Table) and MTT (Memory Translation Table) side by side, with a presented lkey or rkey landing on one MPT entry via a keyed lookup, that entry's arrows fanning out to one or more MTT entries holding its physical-page translations, and a dashed Key Cache box overlapping the top of both tables to mark the working set actually consulted first.](/images/mr/02-the-key-table-block.svg)

## Deep walkthrough

**Why two structures, not one -- the same split a CPU already makes.** The Key Table splits permission-checking from address-translation into two structures for a plausible, analogous reason -- one to answer cheaply and reject fast, one to walk only when the first has already said yes. No public source confirms a real RNIC actually gates MTT this way rather than walking both structures together; this book states it as a reasoned architectural expectation, not a documented hardware order.

> [!WARNING] Analogy
> A CPU doesn't check permission and translate an address in a single step: a PMP-style gate asks "is this access allowed at all," and only once that passes does a TLB-backed page-table walker ask "what physical address does this actually mean." MPT-then-MTT is the same two-question split, applied to a remotely-presented key instead of a locally-issued virtual address.

**Two structures under one name, not one monolithic table.** A real RNIC's memory-registration state splits into two distinct pieces, sourced directly from RNIC-cache literature: a **Memory Protection Table (MPT)**, holding the per-registration metadata [The Memory Region Descriptor](/mr/the-memory-region-descriptor) already covered -- address, length, permissions -- indexed by `lkey`/`rkey`, and a **Memory Translation Table (MTT)**, holding the actual virtual-to-physical page translations an MPT entry points into. A registration spanning many physical pages needs one MPT entry and however many MTT entries cover those pages; a registration backed by one contiguous page needs just one of each. This book uses "the Key Table" as the umbrella term for both together, the same way a reader might see "MTT" used alone in a vendor document -- recognize it as this same territory, not an unexplained separate structure.

**Where this gets checked, and why the failure shape differs from [Queue Pair Management](/qp/)'s QP Table.** Two places ask this table the same question: the local side, resolving an `lkey` from a Work Request this process posted itself, and the remote side, resolving an `rkey` presented in an incoming RDMA operation -- carried in RETH for a WRITE or READ, or in AtomicETH for an Atomic request. [The QP Table](/qp/the-qp-table) covers a QP number that's dense and hardware-assigned, valid the moment it's handed out and reused predictably; a QP number that doesn't correspond to any allocated QP is, in practice, a value nobody legitimate would ever present, because nothing hands out unallocated ones. A key is different in kind: it's a 32-bit value a remote peer holds onto and presents back whenever it next issues an operation, possibly long after registration, and there's no guarantee that value is still current by the time it arrives -- the MR it named may have been deregistered in the meantime. That makes a wrong or stale key here a more consequential failure mode than a QP Table miss: not "this index doesn't resolve," which is roughly what a QP Table faces, but "this index might resolve to nothing, or, if handled carelessly, to a *different* registration that has since reused the same key value." A real implementation has every reason to make key reuse and stale-key rejection an explicit, deliberately-designed part of this table's behavior, precisely because the consequence of getting it wrong is a lookup landing somewhere it shouldn't, not just a miss.

> [!CAUTION] Gotcha
> A stale key isn't guaranteed to fail cleanly the way a stale QP number does. If the entry it once named has since been reused by a different registration, the lookup doesn't miss -- it resolves, successfully, to the wrong memory. Key reuse and stale-key rejection have to be deliberate, designed behavior, not an assumed side effect of the lookup mechanism.

**Addressing mechanism -- an open question, not a documented one.** No public source states whether a presented key indexes this table directly (the way a QP number indexes [The QP Table](/qp/the-qp-table)) or resolves through a hash or other lookup stage. This book describes the table's job as a **keyed lookup** without committing to either shape -- the same conservative move [Queue Pair Management](/qp/) made for QP Table addressing, and for the same reason: a real design choice here isn't publicly documented, and inventing one would be stating a guess as a fact.

**When a lookup doesn't resolve.** A presented key that doesn't map to any currently-valid MPT entry -- never issued, or already retired by a deregistration -- has to fail cleanly rather than resolve to a stale or unrelated entry. [Permission and Bounds Violations](/mr/permission-and-bounds-violations) covers the software-visible consequence (an R_Key violation, the same umbrella status covering a bounds or permission failure); [Mid-Transaction Deregistration](/mr/mid-transaction-deregistration) covers the harder, race-condition version of this same question -- a key that was valid moments ago, presented just as its entry is being torn down.

> [!TIP] Why
> Splitting protection metadata (MPT) from translation entries (MTT) rather than storing one flat per-registration record that inlines both lets a single registration's page count grow without growing the cost of every lookup that only needs the permission/bounds answer, not the full translation chain -- a permission check can stop at the MPT entry; only a lookup that's actually moving data needs to walk into MTT territory. That's the same kind of separation-of-concerns a page-table walker gets from splitting page-table levels out of a single flat mapping, applied here to registration metadata instead of virtual address bits. The cost is exactly what [Key Caching](/mr/key-caching) has to reckon with next: two related structures to keep resident together instead of one, sharing the same finite on-chip working set.

[Key Caching](/mr/key-caching) is next: the on-chip working set that makes touching this table on every operation unnecessary -- and the fact that it's the exact same SRAM [Queue Pair Management](/qp/)'s QP Cache already uses.

## References

- No IETF or IBTA standard governs key-table storage and addressing internals -- it is implementation-specific silicon design, not a wire-format or software-interface contract any standards body defines. This chapter's treatment is grounded in the sources cited in-text above, not a standards document.
