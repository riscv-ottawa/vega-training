# Interrupts and timers

At the end of the last section we built VegaConsole and noticed two problems with it. The first is that `GETCHAR` is a busy-wait: while we sit there waiting for the user to type, the CPU can't do anything else. The second is about structure and decoupling: every line of code we have written so far lives on the same `main`-driven thread, and there is no mechanism for "meanwhile, do this other thing". Both problems have the same fix, and is what we'll be learning next: **interrupts**.

An interrupt is the MCU's way of saying "drop what you're doing, run this code, then put things back the way they were". Anything that can trigger an interrupt (a timer, a button being pressed, a UART byte arriving) becomes a thing the main CPU can react to without having to ask. The catch is that setting one up touches a lot of moving pieces, so prepare yourself for a little bit of a bumpy ride as we work through them in order.

By the end of this section, VegaConsole will look the same to the user, but underneath:

- A timer will toggle a heartbeat LED in the background, regardless of what the REPL is doing.
- `main` will reduce to a single instruction (`wfi`) that puts the CPU to sleep until something happens.
- If you complete the challenge, a user button will pause and resume the heartbeat through a second interrupt - providing a nice visual proof that the foreground main loop and background tasks are now decoupled.

The plan for this section:

1. **[The trap model](./trap-model.md)** is the universal mechanism the CPU uses to handle *any* interruption, whether from a timer, a button, or a *fault* (e.g., an "exception", like an illegal instruction). We'll write a tiny handler from scratch in assembly and use it to gracefully recover from a deliberately broken program.
2. **[EVENT_UNIT and LPTMR](./eventunit-lptmr.md)** is the chip-specific glue: the peripherals that decide *which* interrupt fired and how it reaches the core. In modern RISC-V processors, there are the PLIC or CLINT for this...sadly the RV32M1 has neither, and so this is where the "RISC-V specifies the ISA, not the platform" story really shows up (yet again).
3. **[Blinky, but better!](./blinky-better.md)** rebuilds blinky using a timer ISR. `main` does nothing, the LED still blinks, the REPL still works.
4. **[Challenge](./challenge.md)** asks you to add a user button that is interrupt-driven, it pauses and resumes the heartbeat. Also gets you to explore button debouncing to ensure presses are registered reliably.

If you are following along in Renode, all of this should work in simulation; we'll flag any gaps as they come up.
