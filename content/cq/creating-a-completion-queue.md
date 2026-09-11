---
title: "Creating a Completion Queue"
weight: 3
date: 2026-09-04
publishDate: 2026-09-04
draft: false
description: "Installing a QP needs a state machine. Registering a Memory Region needs a Protection Domain. Creating a Completion Queue needs neither -- just a depth request and, optionally, a way to be told about new entries instead of having to go looking for them."
prev: /cq/the-completion-queue
next: /cq/cqe-generation
---

## Overview

Creating a Completion Queue is the software-to-hardware handoff that produces the ring [The Completion Queue](/cq/the-completion-queue) already described. It's a noticeably thinner handshake than either QP installation or memory registration -- no state machine to drive through, no Protection Domain to belong to, just a depth request and two optional pieces of routing for how software finds out about new entries at all.

## Locator

![The master RNIC block diagram with CQ Storage highlighted and every other block dimmed -- this chapter covers the handoff that produces the ring, not the ring's own shape.](/images/cq/03-creating-a-completion-queue-locator.svg)

The previous chapter's ring has to come from somewhere. This chapter is that somewhere -- what software supplies, what hardware hands back, and what stays fixed versus adjustable afterward.

## Block diagram

![Software requests a minimum depth plus two optional fields (an event channel and a completion vector); hardware returns a CQ handle and the actual allocated depth, always at least the requested minimum -- with no Protection Domain anywhere in the handshake. Below it, a separate resizing call: a current CQ's depth can grow later via an explicit request that must be at least the number of still-unpolled entries, and the actual new depth may round up again.](/images/cq/03-creating-a-completion-queue-block.svg)

## Deep walkthrough

**What creation actually requires -- and what it conspicuously doesn't.** A Completion Queue creation call takes a device context and a requested minimum depth, full stop, as far as mandatory inputs go. [Installing a QP](/qp/installing-a-qp) needs a Protection Domain, and [Registering a Memory Region](/mr/registering-a-memory-region) needs one too -- both are memory-and-permission-scoped objects, and a PD is the boundary that scopes them. A Completion Queue isn't either of those things. It doesn't touch application memory the way a Memory Region does, and it doesn't carry the remote-access permission mask a QP does -- it just holds completion records -- so it has no protection boundary to belong to at all.

> [!NOTE] Concept
> A Completion Queue is the one major RDMA object this book covers that isn't scoped to a Protection Domain. That's not an oversight in the interface -- it's a direct consequence of what a CQ actually is: a reporting structure, not a memory-access object.

**Depth negotiation, in full.** [The Completion Queue](/cq/the-completion-queue) already named the shape of this: software requests a minimum, hardware is free to allocate more, and the actual allocated depth comes back as part of creation's own return value rather than a separate query. Documented directly: a real creation call "creates a CQ with *at least* `cqe` entries," and the caller is expected to read the returned structure's own depth field rather than assume the requested number is what it got. There's no benefit to software over-requesting to compensate for possible rounding -- whatever hardware allocates is already reported back, so the honest number is always available immediately after creation returns.

**The event channel and completion vector -- optional at creation, and this is where polling and events actually fork.** Two more inputs are optional, and both stay unset by default: an event channel, and a completion vector. Neither is required -- a Completion Queue created without either is a purely polling-driven structure, exactly what [The Completion Queue](/cq/the-completion-queue) already described as the baseline case. Supplying an event channel is what makes this CQ capable of generating an asynchronous notification instead of relying entirely on software to keep polling it; a completion vector, when a device supports more than one, is what steers *which* interrupt line that notification actually lands on -- letting different CQs' events spread across different interrupt lines (and typically different CPU cores) rather than all funneling through one. This is where [The WQE and the Send Queue Descriptor](/tx/the-wqe-and-the-send-queue-descriptor)'s `solicit` flag actually cashes out: a solicited-event request only has anywhere to go if the responder's own CQ was created with an event channel in the first place. This Part's own event-arbitration chapter, later, covers what happens once many CQs are all wired up to notify this way at once.

> [!TIP] Why
> Making the event channel optional, rather than mandatory, matches the two real ways an application wants to find out about completions: a latency-sensitive one wants to spin-poll and never sleep, paying CPU cycles instead of wake-up latency; a throughput-oriented one wants to sleep until there's real work, at the cost of a wake-up's latency and the CPU work of servicing an interrupt. Baking a single answer into every Completion Queue would force every application into whichever tradeoff the standard's authors guessed was more common. Making it a creation-time choice pushes that decision to whoever actually knows their own application's access pattern.

**Resizing -- a separate act from creation, not a variation on it.** A Completion Queue's depth doesn't have to stay fixed for its entire life the way [The QP Table](/qp/the-qp-table)'s entries or a Send Queue's ring do -- a real interface documents a distinct resize operation, callable after creation, that requests a new minimum depth for an existing CQ. The one hard constraint: the requested new depth has to be at least the number of entries still sitting unpolled in that CQ right now -- you cannot resize a queue down below data it's already holding that software hasn't consumed yet. Beyond that floor, the same rounding rule from creation applies again: the actual new depth may still come back larger than requested, and has to be read back the same way.

> [!CAUTION] Gotcha
> Resizing is not the same operation as creation, and it isn't how *CQ Overflow* (later in this Part) gets solved reactively either -- it's a deliberate, software-initiated act software has to call explicitly, not something that happens automatically when a CQ starts filling up. Nothing about a Completion Queue grows itself.

**What creation does not fix.** Creating a CQ says nothing about which QPs will ever point at it -- that association happens later, at each QP's own creation or modification, entirely independent of this handshake. A CQ that will end up shared across dozens of QPs and one that will only ever serve a single QP go through the exact same creation call; nothing about creation itself distinguishes the two cases; [Multi-QP Error Arbitration](/qp/multi-qp-error-arbitration) and this Part's own arbitration chapter are where that distinction actually starts to matter. And destruction -- what has to be true about a CQ's outstanding entries and the QPs still pointing at it before it can safely go away -- is this Part's own closing chapter's subject, not this one's.

## References

- No IETF or IBTA standard governs Completion Queue creation parameters, depth negotiation, or the event-channel/completion-vector mechanism -- these are software-interface and implementation-specific silicon design, not a wire-format contract any standards body defines. This chapter's claims (the absent Protection Domain requirement, the depth-rounding behavior, the resize floor, the event-channel/vector split) are grounded in public completion-queue creation and resize documentation, cited in-text above, not a standards document.
