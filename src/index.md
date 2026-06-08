<center>
<h1>
    RISC-V Embedded Systems Training<br/>
    <span style="color: #f17232">VEGA edition</span><br/>
</h1>
</center>

## Overview

This training is provided by [RISC-V Ottawa](https://riscvottawa.ca), as a hands-on introduction to embedded systems development on RISC-V, built around the OpenISA VEGAboard.
Across the sessions, you'll set up a modern containerized toolchain, write and debug firmware for a real RISC-V microcontroller, simulate the same hardware purely in software using Renode, and finally learn how to run applications on top of the Zephyr RTOS.

<center>
    <figure>
        <img src="/img/vegaboard.webp" title="Original source: https://renode.readthedocs.io/en/latest/_images/vegaboard.png" alt="OpenISA VEGAboard" width="40%" >
        <figcaption>OpenISA VEGAboard</figcaption>
    </figure>
</center>

No prior embedded or RISC-V experience is assumed, though comfort with C and the command line will help.

## What will you learn?

The goal of this training material is to teach you the following:

* How to setup a modern containerized embedded systems development environment
* The basics of RISC-V firmware development
  - Focus will be on the [OpenISA VEGAboard (RV32M1-VEGA)](https://github.com/open-isa-org/open-isa.org) development board
* Simulating hardware using [Renode](https://renode.io/)
* The basics of real-time operating systems (RTOS) and [Zephyr](https://www.zephyrproject.org/)
<!--* If time permits, a quick tour of how the [Rust](https://rust-lang.org/) programming language can be used for firmware development and its potential advantages-->

In the end, the hope is that you gain fundamental generalizable knowledge relating to the development of firmware for microcontroller-based systems.
