---
title: "Response Generation"
weight: 9
date: 2026-11-11
publishDate: 2026-11-11
draft: false
description: "Every other chapter in this part follows a packet the local software asked for. This one covers the packets nobody posted a WQE for: the ACKs, NAKs, READ responses and ATOMIC acknowledgments a responder owes, what decides which one goes back, and when it is allowed to wait."
prev: /tx/mid-transaction-tx-reset
metaDescription: "How an RDMA responder decides what to send back: ACKs, NAKs, READ responses and atomic ACKs, implicit acknowledgement, and when a response may wait."
seoTitle: "RDMA Response Generation: ACK and NAK"
---

## Overview

A responder's TX side builds packets no Work Request asked for. Every request that arrives on a Reliable Connection eventually earns an answer: a positive acknowledgment, a negative one, an RDMA READ's data, or an ATOMIC's original value. Response Generation is the logic that decides which of those is owed, which PSN and Message Sequence Number it carries, and whether it has to leave now or may wait and cover several requests at once. It consumes RX PSN Check's verdict on each request and hands a fully specified response to [TX Packet Build](/tx/tx-packet-build), which builds it on the same wire as the QP's own requests.

## Locator

![The master RNIC block diagram with RX PSN Check and TX Packet Build both highlighted, and the "response due" arrow between them lit: the path by which a request accepted or rejected on the receive side becomes a response on the transmit side.](/images/tx/09-response-generation-locator.svg)

[TX Packet Build](/tx/tx-packet-build) named the gap this chapter closes: an ACK or a READ response is built by the same stage as any request, but nothing this part had covered decides when one is due or what goes in its AETH. The "response due" arrow is that decision.

## Block diagram

![RX PSN Check's five verdicts (new valid request, duplicate, PSN sequence error, request error, no Receive WQE) feeding per-QP responder state (expected PSN, the newest PSN an ACK is owed for, the MSN, a NAK latch, and READ and ATOMIC resources). A response selector reads that state, takes a credit count from the Receive Queue, and emits one of four forms toward TX Packet Build: an ACK, a NAK or RNR NAK, an ATOMIC acknowledgment, or a READ response whose payload comes from a DMA read.](/images/tx/09-response-generation-block.svg)

## Deep walkthrough

**What a response is.** IBTA-V1 §9.7.5.1.7 allows exactly three forms: an acknowledge packet for a SEND or RDMA WRITE, an RDMA READ response, and an ATOMIC acknowledgment. All three carry the same four pieces of information: a syndrome saying success or failure, a PSN the requester uses to find the request it answers, a Message Sequence Number, and optionally end-to-end credits. The READ response and the ATOMIC acknowledgment add a payload; the plain acknowledgment never does. A NAK is not a fourth form. It is an acknowledge packet whose AETH syndrome carries a NAK code instead of an ACK (C9-113.a1), even when it replaces a READ response packet.

**The AETH, field by field.** Four bytes: an 8-bit syndrome and a 24-bit MSN (§9.7.5.2). The syndrome's top bit is reserved and zero. Bits 6:5 select the kind: `00` ACK, `01` RNR NAK, `11` NAK, `10` reserved. Bits 4:0 then mean something different for each kind:

| Bits 6:5 | Kind | Bits 4:0 carry |
|---|---|---|
| `00` | ACK | encoded end-to-end credits; `11111` means "no valid credit count" |
| `01` | RNR NAK | the minimum time to wait before retrying (Table 45's timer encoding) |
| `11` | NAK | a NAK code: 0 PSN sequence error, 1 invalid request, 2 remote access error, 3 remote operational error |

The MSN is valid in every one of them, NAKs included (C9-149). [Message Completion and the MSN](/reliability/message-completion-and-the-msn) covers what it counts and how the requester uses it; this chapter only needs the generation rule: one increment per completed new request message, never for a duplicate.

**Which PSN goes in the BTH.** The rule depends on what is being answered, and getting it wrong breaks the requester's matching:

- An ACK for SENDs and WRITEs carries the PSN of the most recent *new* request it covers (C9-95). Because of coalescing, consecutive ACKs need not carry consecutive PSNs.
- A READ response's packets carry the READ request's own PSN, then count up one per packet (C9-96). The requester reserved those PSNs when it sent the request.
- An ATOMIC acknowledgment carries the request's PSN (o9-58); an ATOMIC is a single packet.
- A NAK for a PSN sequence error or an invalid request carries the responder's *expected* PSN (C9-111, C9-121, C9-125), so the requester knows where to resume. A remote access error carries the PSN of the offending request (C9-124), and an RNR NAK the PSN of the request it refuses (C9-113).

**When a response is due: AckReq and coalescing.** For SENDs and WRITEs the spec does not schedule responses at all. It lets one acknowledgment cover several requests (§9.7.5.1.2) and leaves the timing to the responder, bounded by one rule: a valid request with the BTH's AckReq bit set must have a response scheduled, carrying that request's PSN or a later one (C9-116). Without AckReq the responder "may even wait an unbounded time for additional requests." That is why the spec advises a requester to set AckReq on the last packet of whatever it has posted, or risk waiting on an acknowledgment that never comes. The Linux soft-RoCE driver shows both halves of this contract. Its requester sets AckReq on the last packet of every message and on every 64th packet in between (`rxe_req.c`, `RXE_MAX_PKT_PER_ACK`). Its responder answers a SEND or WRITE only when AckReq is set (`acknowledge()` in `rxe_resp.c`), so its coalescing comes entirely from multi-packet messages.

> [!CAUTION] Gotcha
> AckReq has no effect on a READ or an ATOMIC. Both always get an explicit response (§9.7.5.1.6), because the response *is* the operation's result. A responder that coalesced a READ away would leave the requester holding a buffer that never fills.

**A READ response acknowledges more than the READ.** The first packet of a READ response (or its only packet) must carry an AETH, and so must the last (C9-97, C9-98). The two do different jobs. The first one implicitly acknowledges every earlier SEND or WRITE still outstanding at the requester, as a coalesced ACK would. The last one acknowledges the READ itself. Middle packets carry no AETH. When the MSN goes up for the READ is the responder's choice: right after validating it, so the first packet already carries the new value, or once the data has been sent, so only the last does (C9-148). The walk below takes the second option.

Ordering is the subtle part. Because READs are only loosely ordered against later requests, a responder may start executing SENDs and WRITEs that arrived after a READ. But it may not acknowledge them until the READ's response has gone out (C9-101). The reason is on the requester's side: an acknowledgment carrying a PSN later than an outstanding READ implies a NAK for that READ (§9.7.5.1.2, rule 1). An ACK that overtook the READ response would tell the requester to resend a READ that was succeeding.

**Duplicates are answered, not re-executed.** RX PSN Check classifies a request with a PSN behind the expected one as a duplicate ([PSN Check and Out-of-Order Arrival](/rx/psn-check-and-out-of-order-arrival)). Response Generation then owes it a response, and the expected PSN does not move while it is produced (C9-103):

- A duplicate SEND or WRITE is not executed again. It gets an ACK carrying the PSN of the most recently completed new request (C9-105), which tells the requester the responder has already seen everything up to there.
- A duplicate READ is executed again: the data is read from memory a second time, and the response restarts at the duplicate's PSN (C9-107). That only works if the responder still holds the original request's PSN range, which is what the READ resources in the block diagram are for.
- A duplicate ATOMIC is never executed again. If its PSN matches one the responder saved, the saved original value goes back (o9-66). If not, the request is dropped (o9-67). [Atomics](/overview/operation-catalog/atomics) walks this case.

Duplicates are answered in PSN order, oldest first (C9-104, C9-110), and an error while answering one is never reported with a NAK; the duplicate is dropped instead (C9-106, C9-115). A NAK about an old request could be mistaken for a NAK about a new one.

**NAKs, and the latch after a sequence error.** A PSN ahead of the expected one means something was lost. The responder sends one NAK with code 0 and its expected PSN, consumes no Receive WQE, and then ignores every new request until one arrives with exactly that PSN (C9-114, C9-122). It sends no further NAKs in the meantime, while still answering duplicates. The "NAK latch" in the block diagram is that rule as one bit of state. It is set by the NAK and cleared by the matching request. Without it, every packet still in flight behind the lost one would trigger its own NAK, and the requester would rewind once per NAK. [Go-Back-N Retransmission](/reliability/go-back-n-retransmission) covers what the requester does with the one NAK it does get. The other NAK codes follow request errors that RX checks detect: [Permission and Bounds Violations](/mr/permission-and-bounds-violations) covers the remote access error, and an RNR NAK follows a SEND that found no Receive WQE ([The Retry and RNR Timeout State Machines](/reliability/the-retry-and-rnr-timeout-state-machines)).

A READ that fails partway through has its own rule. If the responder hits an error after some response packets have gone out, it sends none of the bad data. The packet that would have carried it goes out as an Acknowledge opcode with the NAK code instead of a READ response, and its PSN names the response packet being refused (C9-100, C9-112).

**Credits: the Receive Queue's say in the AETH.** When the syndrome is an ACK, bits 4:0 carry encoded end-to-end credits: how many more request messages the Receive Queue can take, counted in posted Receive WQEs (§9.7.7.2). An HCA must generate them for any QP without a shared receive queue (C9-150.2.1). A QP on an SRQ cannot know how many WQEs its share of the pool holds, so it must send `11111`, "not valid", instead (o9-95.2.2). The responder can also send credits without anything to acknowledge, by re-sending its most recent positive ACK with an updated credit field. This is called an unsolicited acknowledgment. It can never be made from a NAK, since a duplicate NAK is forbidden. So the Receive Queue feeds this stage, which is the credit-count input in the block diagram.

> [!WARNING] Analogy
> An AckReq bit works like a write-combining buffer's flush. Writes may sit and merge for as long as the buffer likes, until something forces them out. The responder may hold and merge acknowledgments the same way, until AckReq or an operation that needs an explicit answer forces one onto the wire.

Play the whole sequence, from two unacknowledged requests through a READ, a lost packet and the resend:

{{< flow "response-generation" >}}

> [!TIP] Why
> Leaving the ACK schedule to the responder is what makes coalescing cheap. Each ACK costs a slot on the responder's transmit path and a lookup on the requester's receive path. With AckReq, the requester, which knows when it actually needs confirmation, decides how often to pay that cost, and the responder needs no timer of its own to guess. READs and ATOMICs are excluded because their response carries the operation's result, not just a confirmation; there is nothing to coalesce away.

> [!IMPORTANT] Food for Thought
> A responder has executed a READ and three later WRITEs, and the READ's DMA is stalled on a slow memory path. The WRITEs are complete and their requester set AckReq on the last one. What may the responder send now, and what does the requester conclude if a careless design sends the ACK first?

This chapter closes [TX Pipeline](/tx/). [RX Pipeline](/rx/) runs a related set of questions against the receiving side, including the PSN check whose verdicts this chapter turned into responses.

## Interface reference

| Response | BTH opcode | AETH | Payload | BTH PSN |
|---|---|---|---|---|
| ACK | Acknowledge (`0x11`) | syndrome `00` + credits, MSN | none | newest new request covered |
| NAK, sequence error or invalid request | Acknowledge (`0x11`) | syndrome `11` + code, MSN | none | responder's expected PSN |
| NAK, remote access error | Acknowledge (`0x11`) | syndrome `11` + code 2, MSN | none | the offending request's PSN |
| RNR NAK | Acknowledge (`0x11`) | syndrome `01` + timer, MSN | none | the refused request's PSN |
| READ response | First / Middle / Last / Only (`0x0D`–`0x10`) | on First, Last and Only | read data | request PSN, +1 per packet |
| ATOMIC ACK | Atomic Acknowledge (`0x12`) | yes, plus AtomicAckETH | original 64-bit value | the request's PSN |

## References

- InfiniBand Trade Association, *InfiniBand™ Architecture Specification*, Volume 1, Release 1.2.1 -- §9.7.5.1.1 (response PSNs, C9-95, C9-96, o9-58), §9.7.5.1.2 (coalesced acknowledgments and the requester's interpretation rules), §9.7.5.1.3 (READ responses, C9-97 to C9-101), §9.7.5.1.4 (duplicates, C9-102 to C9-110, o9-66, o9-67), §9.7.5.1.5 (NAKs, C9-111 to C9-115), §9.7.5.1.6 (AckReq scheduling, C9-116), §9.7.5.1.7 (response formats), §9.7.5.2 (AETH syndrome, Tables 43 and 44, C9-119 to C9-126), §9.7.7.1 (MSN generation, C9-148, C9-149), §9.7.7.2 (end-to-end credits, C9-150.2.1, o9-95.2.2, unsolicited acknowledgments). Read directly at these section numbers.
- InfiniBand Trade Association, *Supplement to InfiniBand Architecture Specification Volume 1, Release 1.2.1, Annex A17: RoCEv2* (2014) -- carries these transport rules over unchanged.
- Linux kernel soft-RoCE driver, `drivers/infiniband/sw/rxe/` -- `rxe_req.c` sets AckReq on the last packet of every message and every `RXE_MAX_PKT_PER_ACK` (64) packets (`rxe_param.h`); `rxe_resp.c`'s `acknowledge()` answers a SEND or WRITE only when AckReq is set, and every positive AETH it builds uses `AETH_ACK_UNLIMITED` (`0x1f`, the "credits not valid" encoding, `rxe_hdr.h`); `read_reply()` chooses First, Middle, Last or Only by the remaining length against the path MTU.
- No standard governs how a responder holds its pending-ACK state, how large its READ and ATOMIC resources are, or how long it waits before sending an ACK that no AckReq forced; those are stated here as reasoned inference from the rules above.
