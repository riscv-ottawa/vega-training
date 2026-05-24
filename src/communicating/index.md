# Communicating with the world

As you've now seen, blinking an LED for the first time on new hardware actually requires a lot of bring up. But we did it! We now understand our board, a little bit about its SDK, and, most importantly, have our first firmware running! However...our blinky program is really a simple one-bit conversation. Anything more interesting (e.g., a sensor reading, detailed logging, or command from a host) needs the chip to actually talk. This section is about the simplest, oldest, and still most useful way to do that: UART (Universal Asynchronous Receiver-Transmitter).

By the end of this section we'll be able to enter commands into our VEGAboard over a serial terminal, like so:

```
vega> led red on
vega> echo hello there
hello there
vega> cycles
mcycle=12873492 (delta over one csrr-csrr=2)
```

This will be done by developing a simple and tiny read–eval–print loop (REPL), which we'll call the **VegaConsole**
After we develop it, we'll actually keep returning and building on top of it in the following sections.

The plan for this section:

1. **[UART](./uart.md)** will walk through the basics of the protocol itself and the LPUART0 peripheral on the RV32M1. We'll traces a single byte from `PRINTF('A')` down to the store instruction that puts it on the line. Along the way, we'll take two short RISC-V specific detours: reading the `mcycle` CSR to time a UART byte, and looking at our own binary to see where the compressed (`RVC`) instructions are hiding.
2. **[Building VegaConsole](./repl.md)** will wrap a busy-polling receive loop, a line buffer, and a small command table around everything to produce a nice interactive REPL.
3. **[Challenge](./challenge.md)** asks you to add one new command of your own.

As before, you can do everything in Renode if you don't have a board yourself; we'll point out any differences as they come up.
