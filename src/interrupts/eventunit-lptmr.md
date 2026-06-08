# EVENT_UNIT and LPTMR

In the last section we wrote a trap handler that can catch *any* interrupt or exception. But "catch any interrupt" is a bit of an overstatement: the trap fires only when something specific outside the core tells it "an interrupt happened, and here is its number". That something is the chip's interrupt controller, and on RV32M1 the path from "a peripheral has triggered an event" to "your handler responds to it" is longer than you might expect (nothing is ever easy, is it?).

In this section, we'll go over and build the rest of the pipeline. By the end, we'll have a periodic timer interrupt firing every 10 ms, that then land in the trap handler from the previous page, and runs a tiny interrupt service routine (ISR).

This section continues to pair nicely with the `apps/vegaconsole-irq/` example app in the accompanying `vega-quickstart` repository. Feel free to open it up and refer to it as you read along.

## The big picture

If you've read about RISC-V before, you may have come across the standard peripheral called the Platform-Level Interrupt Controller ([PLIC](https://docs.riscv.org/reference/plic/index.html)) or even the non-standard, but common, **CLINT** (for `mtime` / `mtimecmp`). Sadly, the RV32M1 ships neither. Instead, OpenISA went with parts from the PULP family, the same project the RI5CY core comes from.

> [!NOTE]
> **Why not PLIC?**
>
> The RISC-V privileged ISA defines the core's trap CSRs but deliberately does not *mandate* a particular interrupt controller. RISC-V offers several standard ones (PLIC, CLIC, and the newer AIA), all optional, and vendors can also ship something entirely custom as long as it drives those CSRs to spec. PLIC is the best-known of the standard options, but RV32M1 predates these and instead borrowed EVENT_UNIT + INTMUX from the PULP family: tightly coupled to the RI5CY core, smaller than a PLIC but with a different programming model. The next RISC-V chip you pick up may use yet a third design. This is the modularity aspect of RISC-V showing up in a concrete way: same CSRs from the previous page, completely different peripheral wiring around them.

Thus, the parts in the RI5CY that we'll be focusing on are:

- The **EVENT_UNIT**, which sits right next to the core and acts as its interrupt controller. Each IRQ line it presents has its own enable bit, its own priority, and its own number in `mcause`.
- **INTMUX** sits in front of the EVENT_UNIT and fans many peripheral IRQs into a small number of EVENT_UNIT lines. Eight channels, each collecting a group of peripheral sources.

You can learn more about these in Section 3.4 of the RV32M1 reference manual.

So when the timer fires, the signal travels like this before our handler runs:
```
  LPTMR  -->  INTMUX0 channel N  -->  EVENT_UNIT line  -->  core trap
 (peripheral  (one of 8 channels)     (one mcause code)
  IRQ flag)
```

Three handoffs, each with its own enable flag plus the global `mstatus.MIE`. We'll have to make sure all are enabled or our handler will never fire.

## The peripherals at a glance

Each of the blocks mentioned above are just ranges of memory-mapped registers, the same as any other peripheral (e.g., as you saw with LPUART0). From the reference manual:

| Block               | Base         | Size    | Purpose                                                 |
|---------------------|--------------|---------|---------------------------------------------------------|
| EVENT_UNIT (RI5CY)  | `0xE0041000` | `0x88`  | Per-line enable, priority, pending, ISR bit |
| INTMUX0             | `0x4004F000` | `0x200` | Eight channels, each with its own per-source enable mask |
| LPTMR0              | `0x40032000` | `0x10`  | A 16-bit counter with one compare-match interrupt       |

We will next look at them in more detail, starting from the peripheral end (LPTMR) and working inward to the core (EVENT_UNIT), as that's the order the interrupt itself travels.

## LPTMR: a tiny timer

LPTMR (Low-Power Timer) is a simple periodic timer, comprised of a counter that increments off some configured clock, a 16-bit compare register, and one interrupt that fires whenever the counter matches. Chapter 53 of the reference manual covers it in full; we only need four registers:

| Offset | Name  | What it's for                                                |
|--------|-------|--------------------------------------------------------------|
| `0x00` | `CSR` | Control / status: timer enable, interrupt enable, compare flag |
| `0x04` | `PSR` | Clock source select, prescaler, glitch filter                |
| `0x08` | `CMR` | The compare value the counter races toward                   |
| `0x0C` | `CNR` | The counter, read-only                                       |

Let's say we want a tick every 10 ms, or 100 Hz. To keep the arithmetic trivial, we clock the counter from the LPO (a low-power 1 kHz oscillator) and bypass the prescaler (`PBYP`), so the counter advances at exactly 1 kHz: one step per millisecond. To fire `hz` times per second we then just set `CMR = 1000 / hz`, so a 100 Hz tick is `CMR = 10`. So the values the code below sets are the LPO source, the prescaler bypass, and a compare of `1000 / hz`.

```c
#define LPTMR0_BASE 0x40032000u
#define LPTMR0      ((LPTMR_Type *)LPTMR0_BASE)

#define LPTMR_CSR_TEN (1u << 0)   /* timer enable             */
#define LPTMR_CSR_TIE (1u << 6)   /* compare interrupt enable */
#define LPTMR_CSR_TCF (1u << 7)   /* compare flag, w1c        */

void lptmr_init_hz(uint32_t hz) {
    LPTMR0->CSR = 0;                            /* disabled while we configure */
    LPTMR0->PSR = LPTMR_PSR_PCS(1)              /* clock source: LPO, 1 kHz    */
                | LPTMR_PSR_PBYP_MASK;          /* bypass the prescaler        */
    LPTMR0->CMR = 1000u / hz;                   /* fire 'hz' times per second  */
    LPTMR0->CSR = LPTMR_CSR_TIE | LPTMR_CSR_TEN;
}
```

Once we set the `TEN` (timer enable) bit, the counter starts. On compare match, `TCF` gets set, an interrupt request is sent on LPTMR0's output line, and (if the rest of the event pipeline is enabled) we end up in our trap handler.

Note that `TCF` is **write-1-to-clear**. If you forget to clear it inside the ISR, the interrupt re-fires the moment you `mret`, and you have an infinite trap loop (don't ask how I know).

> [!NOTE]
> **Renode workaround.**
>
> The companion code has one extra write that real hardware does not need: an explicit `LPTMR0->CNR = LPTMR0->CMR;` after configuring `CMR`, and the same write again at the end of the ISR. Renode's `LowPower_Timer` model uses a descending counter that gets stuck at 0 after firing, so the counter has to be reloaded explicitly. On real hardware `CNR` is read-only and the LPTMR auto-reloads on compare match, so the extra writes are no-ops. The companion source comments mark each line.

## INTMUX: the fan-in

LPTMR0's IRQ does not go to the EVENT_UNIT directly. It is one of several peripherals wired into a mux channel; these mux channels, in turn, become one EVENT_UNIT line. The reference manual's interrupt table tells you which channel each peripheral lives on; LPTMR0 sits on INTMUX0 channel 0 (keep this in mind for the rest of the chapter).

Each channel has its own little register block, the important one being a 32-bit interrupt enable mask with one bit per source on that channel:

```c
#define INTMUX0_BASE 0x4004F000u
#define INTMUX0      ((INTMUX_Type *)INTMUX0_BASE)

/* Each channel is a sub-block; the SDK exposes them as INTMUX0->CHANNEL[n]. */
INTMUX0->CHANNEL[0].CH_IER_31_0 |= (1u << LPTMR0_SOURCE_BIT);
```

## EVENT_UNIT: the last hop

The EVENT_UNIT is what the core actually sees. Its register block has one bit per IRQ line for enable, plus priority and pending registers; we only care about the enable bit for now. Enabling INTMUX0's channel-0 line at the EVENT_UNIT looks like:

```c
#define EVENT_UNIT_BASE    0xE0041000u
#define EVENT_UNIT_INTPTEN (*(volatile uint32_t *)(EVENT_UNIT_BASE + 0x00))

EVENT_UNIT_INTPTEN |= (1u << INTMUX0_CH0_IRQ);
```


With this final piece, we will have enabled all 3 parts needed to handle interrupts. The last and very final piece is to globally enable this pipeline by setting the machine interrupt enable (`MIE`) in the `mstatus` CSR:
```c
__asm__ volatile ("csrrs zero, mstatus, %0" :: "r"(1u << 3));   /* set MIE */
```

Now the LPTMR compare match will trap into our handler. Finally!

> [!NOTE]
> **`csrrs` vs `csrw`.**
>
> We use `csrrs zero, mstatus, t0` instead of `csrw mstatus, t0` because `mstatus` has many other bits we'd rather not touch. `csrrs` atomically sets only the bits we name and leaves the rest alone; `csrrc` does the same for clearing. Any time you're flipping one specific feature in a shared CSR, you want the read-set or read-clear form. Writing the whole register via `csrw` is for cases like `mtvec`, where you're setting the entire value.

## Dispatching the IRQ

Recalling `trap_handler` from the previous section, we can now deal with the stub we hadn't written:

```c
if (cause & 0x80000000u) {
    irq_dispatch(cause & 0x7FFFFFFFu);
}
```

The low bits of `mcause` are the EVENT_UNIT line number, so the top-level dispatch is just a switch:

```c
void irq_dispatch(uint32_t line) {
    switch (line) {
    case INTMUX0_CH0_IRQ: intmux0_ch0_dispatch(); break;
    /* more lines as we add them */
    default:              spurious_irq(line);     break;
    }
    EVENT_UNIT->INTPTPENDCLEAR = (1u << line);   /* ack the line at the EVENT_UNIT */
}
```

Because the EVENT_UNIT only knows about the channel as a whole, the channel handler has to do its *own* read of the channel's pending register to figure out which peripheral on that channel actually fired:

```c
void intmux0_ch0_dispatch(void) {
    uint32_t pending = INTMUX0->CHANNEL[0].CH_IPR_31_0;
    if (pending & (1u << LPTMR0_SOURCE_BIT)) {
        lptmr0_isr();
    }
    /* other peripherals on this channel would be checked here */
}

void lptmr0_isr(void) {
    LPTMR0->CSR |= LPTMR_CSR_TCF;   /* clear the compare flag, or trap forever */
    g_ticks++;
}
```

> [!NOTE]
>
> The EVENT_UNIT latches requests and keeps re-asserting them to the core until you clear its pending bit too.
> That's the `EVENT_UNIT->INTPTPENDCLEAR = (1u << line)` write in `irq_dispatch` above, done once after the switch so it covers every line.
> If we miss it, the interrupt storms the core even with `TCF` clear, so `main` will hang while the ISR still runs.

The above reads a pending bitmap and then calls the matching handler. These extra layers/invocations to handle a specific peripheral interrupt are the cost of a fan-in interrupt architecture (since the savings are on the hardware side, where there are far fewer lines needed to be wired into the core).


## Tying it together

If we were to create a super minimal program to demonstrate everything we've discussed and shown so far, it would look like this:

```c
volatile uint32_t g_ticks;   /* incremented by lptmr0_isr on every compare match */

int main(void) {
    BOARD_InitDebugConsole();
    install_trap_handler();
    intmux0_enable(LPTMR0_SOURCE_BIT);
    eventunit_enable(INTMUX0_CH0_IRQ);
    set_mstatus_mie();
    lptmr_init_hz(100);      /* start the timer last, after the IRQ path is wired up */

    uint32_t last = 0;
    for (;;) {
        if (g_ticks - last >= 100) {
            PUTCHAR('.');
            last = g_ticks;
        }
    }
}
```

Flash it (or run it in Renode) and one dot per second appears on the serial console. The LPTMR acts as a metronome and the main loop simply checks (very quickly, without blocking) whether it's time to print a dot or not. Next section, we will go even further and let the CPU go to sleep entirely when there is nothing to do.

## TLDR

- RV32M1 is old and has neither a CLINT nor a PLIC. Interrupts go through the so-called EVENT_UNIT (core-local controller) fed by an INTMUX (peripheral fan-in).
- Three enable flags must all be set for a peripheral IRQ to reach your handler: the peripheral's own interrupt-enable, the INTMUX channel's per-source mask, and the EVENT_UNIT's per-line enable. Plus `mstatus.MIE` globally.
- LPTMR is a 16-bit counter with one compare-match interrupt. Four registers (`CSR`, `PSR`, `CMR`, `CNR`) are all we need.
- Dispatch happens in two layers: a switch on `mcause` to find the EVENT_UNIT line, then a read of the INTMUX channel's pending register to find the peripheral.
- Always clear the peripheral's interrupt flag inside the ISR. LPTMR's `TCF` is write-1-to-clear. If you forget it, you are trapped forever! Similarly, the EVENT_UNIT pending bit (`INTPTPENDCLEAR`) must also be cleared or the interrupt storms even with `TCF` clear.
