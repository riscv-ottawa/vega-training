# [Training] RISC-V Embedded Systems Training (VEGA edition)

## Development Board

* [OpenISA VEGAboard (RV32M1-VEGA)](https://github.com/open-isa-org/open-isa.org)

## Building the book

The website is built using [mdBook](https://github.com/rust-lang/mdBook).

The folder of primary relevance is [src](./src).
This is the content of the website in markdown.

Build the book using the command `mdbook build`.
This command will output the generated content (HTML, CSS, etc) to `./book/`.
For hosting properly for others, you should serve these files using a simple HTTP server.

For local testing, run `mdbook serve` to serve the contents locally.
This command will keep running, watching for changes and regenerating the site anytime a change is made to the source material.

## Resources

* [OpenISA VEGAboard (RV32M1-VEGA)](https://github.com/open-isa-org/open-isa.org)
* [RV32M1 SDK (RISCV)](https://github.com/open-isa-rv32m1/rv32m1_sdk_riscv)
* [OpenOCD](http://openocd.org/)
* [Zephyr - OpenISA VEGAboard](https://docs.zephyrproject.org/latest/boards/openisa/rv32m1_vega/doc/index.html)
* [FreeRTOS - RISC-V RV32M1 VEGAboard Demo](https://compiler.freertos.org/Documentation/02-Kernel/03-Supported-devices/04-Demos/Others/RTOS-RISC-V-Vegaboard_Pulp)
* [The Embedded Rust Book](https://rust-embedded.github.io/book/)
* [Rust Embedded MB2 Discovery Book](https://docs.rust-embedded.org/discovery-mb2/index.html)
* [Rust - HAL for the RI5CY core of RV32M1](https://github.com/rv32m1-rust/rv32m1_ri5cy-hal)
