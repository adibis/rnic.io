---
title: rnic.io
type: docs
layout: hextra-home
seoTitle: "RNIC: RDMA NIC Architecture, Block by Block"
metaDescription: "A datasheet-depth guide to RDMA NIC hardware: RoCEv2, Queue Pairs, memory registration, reliability, congestion control and DMA, with playable walks."
---

<div class="rnic-home-hero rnic-home-bleed not-prose">
<div class="rnic-home-inner">
<p class="rnic-home-badge">A hardware-level guide to RDMA NICs</p>
<h1>RDMA NIC architecture, block by block</h1>
<p class="rnic-home-lede">Most writing on RDMA stops at &ldquo;the hardware handles it.&rdquo; This book opens the RNIC: every table, cache, state machine and pipeline stage between a Work Request and the wire.</p>
<div class="rnic-home-cta">
<a class="rnic-home-btn" href="overview/">Start reading &rarr;</a>
<span class="rnic-home-stats">{{< home-stats >}}</span>
</div>
</div>
</div>

<section class="rnic-home-section not-prose">
<div class="rnic-home-label-row">
<h2 class="rnic-home-label">Contents</h2>
<p class="rnic-home-sublabel">In reading order, from what RDMA asks of a NIC to the last address translation</p>
</div>

{{< home-parts >}}
</section>

<section class="rnic-home-band rnic-home-bleed not-prose">
<div class="rnic-home-inner">
<div class="rnic-home-label-row">
<h2 class="rnic-home-label">The whole adapter</h2>
<p class="rnic-home-sublabel">Every block is a chapter. Click one to open it.</p>
</div>
<div class="master-diagram-wrap rnic-themed-svg rnic-home-map">
{{< inline-svg "static/images/overview/00-master-diagram-linked.svg" "The RNIC master block diagram: every hardware block between a Work Request and the wire, each one a link to the chapter that covers it" >}}
</div>
</div>
</section>

<section class="rnic-home-section not-prose">
<div class="rnic-home-label-row">
<h2 class="rnic-home-label">Step-through walks</h2>
<p class="rnic-home-sublabel">Each packet's headers, each block's state, and a question to answer before the next step</p>
</div>

<div class="rnic-home-walks">
{{< home-walk page="/overview/one-transaction-start-to-finish" flow="one-transaction" title="One RDMA Write, start to finish" >}}A WRITE with Immediate through both adapters, both hosts and a switch.{{< /home-walk >}}
{{< home-walk page="/overview/one-all-reduce" flow="allreduce" title="A ring all-reduce" >}}Four hosts summing their buffers, hop by hop, in the RDMA operations NCCL issues.{{< /home-walk >}}
{{< home-walk page="/reliability/go-back-n-retransmission" flow="go-back-n" title="A lost packet" >}}One packet dropped, a NAK, and go-back-N resending three.{{< /home-walk >}}
{{< home-walk page="/qp/qp-caching" flow="qp-cache-miss" title="A QP Cache miss" >}}A miss, a dirty write-back, and the evicted QP missing again moments later.{{< /home-walk >}}
{{< home-walk page="/congestion/cnp-generation" flow="incast" title="Two senders, one port" >}}ECN marks, CNPs and two rate cuts at a congested switch port.{{< /home-walk >}}
{{< home-walk page="/dma/the-gather-path" flow="sge-gather" title="Gather and scatter" >}}One SEND read from three buffers and written into two.{{< /home-walk >}}
</div>
</section>

{{< recent-posts >}}
