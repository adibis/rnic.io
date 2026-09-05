---
title: About
---

Most public RDMA/RoCEv2 material treats the RNIC as a black box: "the hardware handles PSN tracking, address translation, retries." That's the right level for a networking overview, and the wrong level if you're the one who has to design, verify, or debug that hardware. This site opens the box -- every one of those bullets becomes a chapter with a pipeline stage, a state machine, a caching structure, and a failure mode.

## Who's writing this

**rnic.io** is written by Aditya Shevade, a design verification engineer with 15 years in the field, currently at Meta. His background spans FPGA prototyping through ASIC and SoC verification methodology, with graduate work at USC in computer architecture and VLSI design. Before DV, he wrote technical articles for Linux Format, Linux Magazine, and Open Source For U.

Get in touch: [aditya.shevade@gmail.com](mailto:aditya.shevade@gmail.com) · [github.com/adibis](https://github.com/adibis) · [LinkedIn](https://www.linkedin.com/in/adityashevade)

rnic.io is the RDMA-hardware-architecture companion to [chipdv.io](https://chipdv.io), which covers the verification side of chip design: CSR verification, memory allocation, testbench architecture, and firmware.

## Colophon

Built with [Hugo](https://gohugo.io) and the [Hextra](https://github.com/imfing/hextra) theme -- the same stack as chipdv.io. Diagrams are hand-built SVGs lettered in Tecnico Fino.
