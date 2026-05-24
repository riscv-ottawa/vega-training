# Challenge

Pick **at  least** one of the three commands below and add it to VegaConsole.
Each one is small (a dozen or two lines).
Bonus points for picking the one that scares you most.

Either path works for showing your result: a Renode session, or a real board with a serial terminal.

## Option A: `blink <hz>`

Take a frequency in hertz and busy-loop the red LED at that rate. So `blink 2` toggles twice a second, `blink 10` ten times a second.

What you'll learn:

- A second use of the `delay()` pattern from blinky, this time parametrized.
- How `mcycle` makes a *much* better timer than counting NOPs. (At 48 MHz, you can derive cycles-per-half-period from the requested Hz and just spin until `csr_mcycle()` advances by that much. No more guessing.)

The trap: while `blink 5` runs, the REPL is frozen. You can't type the next command, can't even Ctrl-C. That's fine for now, because the frozen REPL problem is exactly what we'll fix in the next section.

> [!NOTE]
> **Hint:** `int hz = atoi(argv[1]);` is fine for parsing. For the timing, compute `cycles_per_half = 48000000 / (2 * hz);` and busy-wait on `mcycle` until that many cycles have passed.

## Option B: `dump <addr> <len>`

Print `<len>` bytes of memory starting at `<addr>`, in classic hex-dump format: 16 bytes per line, address on the left, hex in the middle, ASCII on the right. Try it on `0x00000000` (the start of flash, where the vector table lives) and on `0x20000000` (the start of RAM).

What you'll learn:

- Parsing hex addresses out of `argv` (`strtoul(argv[1], NULL, 16)`).
- Reading memory through a `volatile uint8_t *` so the compiler doesn't optimize it away.
- That the chip memory layout is exactly the way the linker script said it would be.

The trap: feed it a bogus address like `0xDEADBEEF` and the chip will fault on the load and lock up. There's no recovery, because we have not written a trap handler yet. Next section we will, and then `dump 0xDEADBEEF` will print the offending address and gracefully return to the prompt.

> [!NOTE]
> **Hint:** Strict-aliasing rules are why you want `volatile uint8_t *p = (volatile uint8_t *)addr;` rather than casting through `int *`. Reading one byte at a time is slow but works on any address and is safer; doing word-at-a-time is faster but may fault on misaligned addresses.

## Option C: `csrdump`

Print a small fixed list of CSRs by name and value. A good starter set is `mcycle`, `minstret`, `mhartid`, `mvendorid`, `marchid`, `mimpid`, `misa`. Make the output look like:

```
vega> csrdump
mhartid    = 0x00000000
mvendorid  = 0x00000000
marchid    = 0x00000004
mimpid     = 0x00008000
misa       = 0x40901105
mcycle     = 0x00d4afc0
minstret   = 0x0089e0a3
```

What you'll learn:

- The inline-assembly pattern for reading any CSR (hint: see the `csr_mcycle` helper from a previous section).
- If you spend some extra time, you'll hopefully learn how to decode the values of these CSRs. For example, `misa` has the high two bits encode XLEN, the low 26 are a bitmap describing the supported RISC-V extensions; bit 0 = `A`, bit 2 = `C`, bit 8 = `I`, bit 12 = `M`, etc. RI5CY should report I + M + C set.
- That CSRs are not just for cycle counters; they are *the* mechanism RISC-V uses to expose machine state.

The trap: there isn't one, easy peasy. But everything you build here will be very useful for the next section, when we start looking at CSRs (`mstatus`, `mie`, `mip`, `mtvec`, `mcause`, `mepc`) much more.

> [!NOTE]
> **Hint:** Create a small macro!
> ```c
> #define CSR(name) \
>     ({ uint32_t v; __asm__ volatile ("csrr %0, " #name : "=r"(v)); v; })
> ```
> `CSR(mcycle)` then expands to a one-instruction read. The `#name` stringifies the macro argument so the assembler sees the right CSR name.
