# Blinky, but better!

The very first program in this book was [blinky](../firmware/blinky.md), the hello world of embedded systems. This simple program blinked an LED by poking some GPIO registers and used a hand-tuned busy-wait NOP loop for the delay between each blink. It worked...but, now that you know a thing or two about interrupts, it's easy to see how poorly it was implemented. First, the CPU was pinned at 100% doing barely anything useful. Second, the delay duration was complete guesswork tuned to one clock setting. And finally, there was no way for `main` to do anything else. Luckily, we now have the tools improve this.

This section goes over rebuilding blinky using timers and interrupts. You can find the full example code at `apps/blinky-better/` in the accompanying `vega-quickstart` repository, we'll also integrate some of this back into `apps/vegaconsole-irq`. Feel free to open up and refer to them as you read along.

## The new architecture

Our old blinky looked something like this:
```c
while (1) {
    delay();
    led_toggle();
}
```

The `delay()` function was not very smart, it just continuously incremented a counter. This was imprecise (we guessed the timings for the delay based on the currently configured clock) and, worse, entirely blocked the CPU from doing anything else useful.

The new blinky will look something like this:
```c
void lptmr0_isr(void) {
    LPTMR0->CSR |= LPTMR_CSR_TCF;
    led_toggle();
}

int main(void) {
    init_everything();
    while(1) {
        __asm__ volatile ("wfi");
    }
}
```

This is the same blinky functionality, but with completely different control flow. This time, the hardware owns the timing and the CPU sleeps between events. `main()` does no work whatsoever.

## The new program

Here is roughly what the full program looks like in detail:
```c
#include "fsl_common.h"
#include "board.h"

#define LED_GPIO  GPIOE
#define LED_PIN   29u

static inline void led_toggle(void) {
    LED_GPIO->PTOR = 1u << LED_PIN;
}

volatile uint32_t g_blinks;

void lptmr0_isr(void) {
    LPTMR0->CSR |= LPTMR_CSR_TCF;     /* very important: clear before mret */
    led_toggle();
    g_blinks++;
}

int main(void) {
    BOARD_InitDebugConsole();
    led_init();                       /* same as before */

    install_trap_handler();
    intmux0_enable(LPTMR0_SOURCE_BIT);
    eventunit_enable(INTMUX0_CH0_IRQ);
    set_mstatus_mie();
    lptmr_init_hz(2);                 /* start last; toggle 2x/s = 1 Hz blink */

    PRINTF("blinky, but better. main going to sleep.\r\n");
    while(1) {
        __asm__ volatile ("wfi");
    }
}
```

The LED once again blinks at 1 Hz, and this time `main()` never executes another instruction after the `wfi`.
If we were to open GDB and you stop the chip, we'd find the program counter parked on the `wfi` instruction.

## Wait, what is `wfi`?

The `wfi` instruction stands for "wait for interrupt". In most cases, if you use `wfi` the core stops fetching instructions and most of its clocks turn off, dropping power consumption to a small fraction of what it was. The next interrupt wakes the core, the trap path we detailed previously runs, and then execution resumes at the instruction right after `wfi`. Because we put `wfi` in a loop, we immediately put the core back to sleep once the interrupt has been serviced.

Note that this is not a halt. The chip is not turned off. This is just the standard way to idle. It is used to do nothing, wait for the next event, do nothing again. You'll see this on every MCU you come across going forward.

> [!NOTE]
> **`wfi` is a hint, not a guarantee.**
>
> The RISC-V spec lets the underlying implementation treat `wfi` as a plain no-op if it wants to.
> On RI5CY it actually sleeps, but simpler cores (e.g., those for education) might just spin instead.
> Either way, using it is probably the correct thing to do, as it signals that we only want to make forward progress when there is an interrupt to wake on.

## Sharing data with an ISR

One final, but crucially important, detail we need to cover is the mechanism used to share data between the main loop and interrupt service routines.

The main difficulty is that an ISR runs asynchronously. It can fire between any two instructions of `main`, change a variable, and return, all without `main` ever calling it. The compiler, on the other hand, optimizes `main` as if it were the only code that runs. It has no idea the ISR exists, let alone that the two share the same memory. If we don't do anything about this, the compiler will make assumptions that are perfectly valid for ordinary single-threaded code but wrong the moment an interrupt gets involved.

This is why `g_blinks` is declared `volatile`. The `volatile` keyword tells the compiler that the variable can change randomly at anytime, and so don't cache it in a register! To see why that matters, imagine a loop that waits for the ISR to count ten blinks:

```c
while (g_blinks < 10) { /* wait for 10 blinks */ }
```

Nothing inside this loop changes `g_blinks`, so the compiler concludes the value is stable. It loads `g_blinks` into a register once and then spins forever comparing that one cached copy against 10. Meanwhile an ISR may be happily incrementing the real variable in memory, but `main` never looks at memory again, so the loop never exits. Marking `g_blinks` as `volatile` forces a fresh load from memory on every iteration, which is exactly the behaviour needed.

Note that `volatile` fixes visibility/access, but it is *not* an atomicity guarantee! Atomicity is about whether an operation completes in one indivisible step or can be interrupted partway through. On RV32, loading or storing a 32-bit variable like `g_blinks` is a single instruction, so a reader can never catch it half-updated. A 64-bit variable on a 32-bit system could be another story: it may take two instructions to load, and if the ISR fires in between, `main` can read the low half of the new value stitched to the high half of the old one. The same hazard shows up whenever both `main` and an ISR write the same variable, because even `g_blinks++` is really three steps (load, add, store), and an interrupt landing in the middle of them will quietly lose an update. Our blinky is safe only because the ISR is the sole writer and `main` merely reads, but keep the rule in mind: once a value is updated from both sides, `volatile` alone is not enough. We won't be covering the solution to problems like this in this book, but reading into mutexes and other synchronization mechanisms would be the next step.

> [!NOTE]
> **A word on `fence`**
>
> RISC-V has `fence` instructions for *memory ordering*.
> You can read more about it [here](https://docs.riscv.org/reference/isa/unpriv/mm-eplan.html#mm-fence).
> On a single-issue, in-order, single-core chip like RI5CY, you can usually get away without it.
> This is because by the time `mret` returns from an ISR, every write the ISR did is visible to `main`, and vice versa. When dealing with more complex cores (out-of-order, multi-core, or with separate I/O ordering domains) `fence` is important.
> No need to worry about for what we're doing.

## Upgrading VegaConsole

As one last way to drive this knowledge home, let's add this blinky code as a background heartbeat to our upgraded VegaConsole app. We'll use the same LPTMR ISR and `g_blinks` counter.
The important thing to note is that the REPL will still remain responsive to reading characters, even while the heartbeat LED we add blinks. If you type while a blink is in progress, the toggle just takes a handful of cycles and control returns to the REPL before you've finished pressing the next key.

If you don't believe this, then add a `slowcmd` command like below that runs a deliberate 2-second busy-loop:
```c
static int cmd_slow(int argc, char **argv) {
    for (volatile uint32_t i = 0; i < 20000000; i++) { }
    return 0;
}
```

Run `slowcmd` to freeze the REPL for two seconds, but keep an eye on the LED. It *keeps blinking the whole time*! The foreground is stuck; the background isn't...and this is all on a single CPU core!

## TLDR

- A timer-ISR blink replaces the busy-wait `delay()` of old blinky. Same visible behaviour, fundamentally different control flow.
- When fully relying on interrupt-based events, `main()` can be reduced to `while (1) { wfi; }`. The CPU sleeps between events.
- Shared variables between ISR and `main()` need `volatile` (at the very least) so the compiler doesn't optimize/cache access. `volatile` is not the same as atomic.
- `fence` exists for memory ordering. We don't need it on our chip, but you will need it in the future if targeting fancier cores.
- The main thing we achieved is: decoupling. The logic in an ISR survives any amount of foreground blockage. Every later layer and abstraction you see going forward (RTOS threads, drivers, schedulers) will likely be built using this same property.
