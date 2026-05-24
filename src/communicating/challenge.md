# Challenge

Pick **at  least** one of the three commands below and add it to VegaConsole.
Each one is small (a handful of extra lines of C).
Bonus points for picking the one that scares you most.

> [!WARNING]
> If running purely in simulation, avoid option A and C (since they require you to use CSRs that are not modeled in Renode).

## Option A: `blink <hz>`

Take a frequency in hertz and busy-loop the red LED at that rate. So `blink 2` toggles twice a second, `blink 10` ten times a second.

What you'll learn:

- A second use of the `delay()` pattern from blinky, this time parametrized.
- How a cycle counter makes a *much* better timer than guessing and counting NOPs.

The trap: while `blink 5` runs, the REPL is frozen. You can't type the next command, can't even Ctrl-C. That's fine for now, because the frozen REPL problem is exactly what we'll fix in the next section.

> [!NOTE]
> **Hint:** `int hz = atoi(argv[1]);` is fine for parsing.
>
> For the timing, compute `cycles_per_half = 48000000 / (2 * hz);` and busy-wait on `csr_cycles()` (PULP `pccr0` at CSR `0x780`) until that many cycles have passed. The `2 * hz` is because one blink is two LED toggles (on, then off), so you need to flip the pin twice per period: at `hz` blinks/sec the time between flips is `1 / (2 * hz)` seconds, which at 48 MHz works out to `48000000 / (2 * hz)` cycles. Remember to call the `enable_perf_counters()` function we learn about earlier once at startup.

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

Print a small fixed list of CSRs by name and value. Something like:

```
vega> csrdump
mstatus  = 0x00001808
mtvec    = 0x000fff00
mepc     = 0x00000000
mcause   = 0x00000000
pccr0    = 0x1a870094
pcer     = 0x00000001
pcmr     = 0x00000001
privlv   = 0x00000003
```

> [!WARNING]
> **Pick CSRs that this chip actually implements.** The RV32M1's RI5CY is an older core and sadly does *not* have any of the standard RISC-V CSRs (`mhartid`, `mvendorid`, `marchid`, `mimpid`, `misa`) or the standard timing counters (`mcycle`, `minstret`). Trying to read those does not cause an error, but just returns 0.
>
> The full list of CSRs this core actually implements lives in **Section 3.1.3.6 of the RV32M1 Reference Manual**; the trap-handling CSRs (`mstatus`, `mtvec`, `mepc`, `mcause`), the PULP performance counters (`pccr0`, `pcer`, `pcmr`), and `privlv` are all there (as shown above) and can be added.
>
> The "all zeros means it doesn't exist" failure mode mentioned above also means there is no compile-time or runtime error telling you you've picked an unimplemented CSR...so make sure to cross-reference with the manual.

What you'll learn:

- The inline-assembly pattern for reading any CSR (hint: see the `csr_cycles` helper from a previous section). Note that the helper used a numeric address (`0x780`) rather than a symbolic name (see the hint below).
- That `privlv` reads back `3` confirms the CPU is in machine mode, which is where the RISC-V privileged spec says you should start after reset.
- `mtvec` is the base address of the trap vector; right now it points wherever the boot ROM or startup code left it because we have not installed our own handler yet.
- `mstatus` has bits for the global interrupt enable (`MIE`, bit 3) and the previous privilege fields used on trap entry.
- That CSRs are not just for cycle counters; they are *the* mechanism RISC-V uses to expose machine state.

The trap: not all CSRs you might expect exist on this chip, and the ones that don't exist just read as 0 rather than trap. There's no way to tell "real zero" from "unimplemented zero" from software alone.

> [!NOTE]
> **Hint:**
> Here is a macro to help you with this challenge. It takes the numeric address of a CSR, reads the value using the `csrr` instruction via inline assembly, then returns the value of the CSR:
> ```c
> #define CSR(addr) \
>     ({ uint32_t v; __asm__ volatile ("csrr %0, " #addr : "=r"(v)); v; })
> ```
>
> Remember: look each CSR's address up in Section 3.1.3.6 of the reference manual.
