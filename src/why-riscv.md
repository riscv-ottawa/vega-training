# Why RISC-V?

[RISC-V](https://riscv.org/technical/specifications/) is an open, royalty-free Instruction Set Architecture (ISA) originally developed at UC Berkeley in 2010. Unlike ARM or x86, anyone can implement a RISC-V processor without licensing fees or legal agreements, and anyone can read the official specification without signing an NDA.

That openness has turned RISC-V into something much larger than a research or education project. It has crept into virtually every corner of industry. NVIDIA alone ships [over a billion RISC-V cores](https://media.defcon.org/DEF%20CON%2033/DEF%20CON%2033%20presentations/Adam%20Zabrocki%20Marko%20Mitic%20-%20How%20to%20secure%20unique%20ecosystem%20shipping%201%20billion%2B%20cores.pdf) embedded throughout its hardware stack, a number that sounds absurd until you see how pervasively they use it across GPUs, embedded controllers, and security subsystems. If you use a modern SSD, GPU, or cloud service, you may be running RISC-V without even knowing it.

The ecosystem that has grown around the specification is what makes RISC-V more than just another ISA. [RISC-V International](https://riscv.org/), the non-profit Swiss foundation that governs the standard, coordinates hundreds of member organizations, from chip vendors and hyperscalers to universities and independent engineers, all of whom have an equal seat at the table. The specification itself evolves in the open on public mailing lists and GitHub repositories, in contrast to the closed processes behind proprietary ISAs.

Around that core specification, a layer of open-source silicon has emerged. The [OpenHW Group](https://www.openhwgroup.org/) develops production-quality, verification-heavy RISC-V cores (the CV32E40P and CVA6 families among them) that companies can integrate into real products. [lowRISC](https://www.lowrisc.org/), a non-profit based in Cambridge, maintains [Ibex](https://github.com/lowRISC/ibex), a small 32-bit core that has found its way into everything from educational FPGA boards to the security subsystems of large-scale infrastructure. These are not toy designs; they are built, tested, and shipped in production hardware (for example, in [Google Chromebooks](https://opensource.googleblog.com/2026/03/opentitan-shipping-in-production.html)).

The security research community has also adopted RISC-V as its testbed of choice, precisely because the ISA is hackable in the best sense: you can modify it, extend it, and tape out your ideas. The [RISC-V Platform Security Model Specification](https://github.com/riscv-non-isa/riscv-platform-security-model) is a framework for hardware-level security primitives, covering physical memory protection, trusted execution environments, and attestation. At the research frontier, CHERI (Capability Hardware Enhanced RISC Instructions), and specifically [CHERIoT](https://cheriot.org/) (its adaptation for microcontrollers), is implemented as a RISC-V extension. CHERIoT runs on real hardware today, including on the [Sonata FPGA board](https://www.sunburst-project.org/), which is built around the Ibex core. That same Ibex core powers [Google's OpenTitan](https://opentitan.org/) root-of-trust chip.

So when you learn RISC-V, you are not learning a niche curiosity. The ISA we will be using in this training relates to research, silicon that ships in billions of devices, and secures real systems.

At a high-level, RISC-V is:

1. Open: a royalty-free Instruction Set Architecture (ISA) originally developed at UC Berkeley in 2010 and now governed by an open and transparent non-profit foundation (RISC-V International).
2. Community driven: primarily driven and evolved through open-source specifications and implementations, allowing individuals (like us) and industry to freely contribute and continually advance it.
3. Modular: The RV32I for 32-bit and RV64I for 64-bit base instruction sets are the minimum for each implementation (our development board builds on RV32I, adding the M and C extensions, i.e. RV32IMC). After that, there are over 100 ratified extensions to pick and choose from when designing real-world hardware.

> [!NOTE]
> You may be thinking that writing software for an architecture supporting 100+ extensions would be a comparative nightmare.
> To avoid such nightmares, RISC-V has developed something called [profiles](https://riscv.atlassian.net/wiki/spaces/WPMU/overview) which "are named groupings of standard processor ISA bases plus extensions (each identified as Mandatory or Optional)".

RISC-V was shaped by decades of lessons from earlier architectures (MIPS, SPARC, Alpha, ARM), and its designers made deliberate choices that directly affect how simple our hardware implementation can be. The [RISC-V Reader](http://www.riscvbook.com/) summarizes this beautifully in its very first chapter by considering many aspects important to ISA design such as cost, simplicity, performance, isolation of architecture from implementation, room for growth, program size, and ease of programming. If you want to learn more about the RISC-V ISA on your own as we progress, it is highly recommended to pick up a copy of the RISC-V Reader!

All-in-all, RISC-V is here to stay, it is open, and growing fast. We hope this motivates you to learn and continue this training with us!
