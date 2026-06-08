# Challenge

We've built up VegaConsole to now contain a nice heartbeat LED that blinks entirely on its own, driven by the LPTMR ISR while `main()` runs the REPL. This challenge adds one more interrupt source, the user button on the VEGAboard, and uses it to control that heartbeat.

The point of this exercise is to get you to wire up a peripheral and its interrupt source yourself. You've already wired one peripheral all the way to the core, doing this second one yourself will cement the pattern in your noggin.

The challenge comes in two parts:
* Part 1 makes a button press pause and resume the heartbeat.
* Part 2 guides you to fix the bug you will hit the moment you try Part 1 on real hardware :D.

Open up `apps/vegaconsole-irq` and let's get started!

## A different kind of interrupt line

Actually, there is one difference worth knowing about before you start (make sure to read this, it will save you time). For the LPTMR interrupt, we saw that it did not reach the core directly. Instead, it fanned in through INTMUX channel 0, which then drove EVENT_UNIT line 24 (`INTMUX0_0_IRQn`). That is why the timer needed three enables: its own `TIE`, the INTMUX channel mask, and the EVENT_UNIT line.

However, the button you'll be asked to configure is wired differently. On the VEGAboard the user button (SW2) sits on PORTA pin 0, and PORTA's interrupt is `PORTA_IRQn`, which is line 18. Lines 0 through 23 connect *straight* to the EVENT_UNIT, with no INTMUX in between (the fan-in only covers lines 24 to 31). So there is no channel mask to set and no channel pending register to read. Recall the pipeline diagram from [EVENT_UNIT and LPTMR](./eventunit-lptmr.md): the button takes the short path.

```
  LPTMR0  -->  INTMUX0 ch0  -->  EVENT_UNIT line 24  -->  core
  PORTA   ------------------>    EVENT_UNIT line 18  -->  core
```

In practice this means the button needs *fewer* moving pieces than the timer did: configure the pin to interrupt, enable its one EVENT_UNIT line, and handle it. No `intmux0_enable` call at all. Easy peasy.

## Part 1: pause and resume

The heartbeat toggle lives in `lptmr0_isr`. Right now it toggles the LED unconditionally on every half-period. We want a flag that gates it:

```c
volatile bool g_beat_paused;

void lptmr0_isr(void) {
    LPTMR0->CSR |= LPTMR_CSR_TCF_MASK;   /* ack as before */
    LPTMR0->CNR = LPTMR0->CMR;           /* Renode reload, as before */
    g_ticks++;                           /* keep counting regardless */
    if (!g_beat_paused && (g_ticks % HEARTBEAT_HALF_TICKS) == 0u) {
        heartbeat_toggle();
    }
}
```

Notice `g_ticks` still increments while paused. Only the LED toggle is gated, so timing stays consistent and the counter keeps running for anything else that needs it (including Part 2).

For the button, you need to:

1. **Configure the pin:** set PORTA pin 0 as a digital input and configure a *falling-edge* interrupt on it (the button pulls the line low when pressed):
    - Hint: refer to `fsl_port.h` and `fsl_gpio.h` from the SDK. The exact pin, port, and IRQ for SW2 are also in the SDK's GPIO input-interrupt example, you can add the matching macros to your `board.h` rather than hard-coding `0` everywhere.

2. **Enable the line at the EVENT_UNIT:** This is the whole enable path for a direct line (no INTMUX step, because the button does not go through INTMUX.)

3. **Dispatch:** add a case to the `irq_dispatch` switch you already have. Because PORTA is a single-source line, not a shared INTMUX channel, it calls its ISR directly, with no channel pending-register read:
   - Note: the `EVENT_UNIT->INTPTPENDCLEAR` ack at the bottom of `irq_dispatch` already covers every line, so you do not need to touch that.

4. **Write the ISR handler:** the ISR is tiny, it should just clear the appropriate interrupt flags and flip `g_beat_paused`.
    - Note: PORTA's pin-interrupt flag (`ISFR`) is write-1-to-clear, exactly like the LPTMR's `TCF`. If you forget to clear it, the handler re-fires the instant you return, trapping the core forever. Don't forget it!

## Part 2: debounce

If you try Part 1 on real hardware, you will likely notice right away that the button is unreliable. Sometimes a press may pause the heartbeat, sometimes it seems to do nothing, and occasionally you might catch the LED flickering for an instant as you press.

Believe it or not, the cause is mechanical...yuck! Unless you have a really fancy button, a button does not make one clean contact; it bounces, opening and closing several times over the first few milliseconds. Each bounce is another falling edge, so a single press fires your ISR several times in quick succession. Since the ISR toggles `g_beat_paused`, an even number of bounces lands you right back where you started, which is why some presses appear to do nothing.

The fix is to ignore edges that arrive too close together to be separate presses. You already have a clock for this: `g_ticks`. Record the tick of the last press you accepted, and reject any new edge that lands within a short window, say 20 ms (two ticks at the 100 Hz rate):

Remember to always clear the interrupt flag first, even for the bounces you are about to discard, or you are back to the trap-forever problem.

> [!NOTE]
> There is an interesting thing here to note now. We have `porta_isr` that now *reads* `g_ticks`, and `lptmr0_isr` who *writes* it.
> Two interrupt handlers sharing a variable is exactly what we talked about in [Sharing data with an ISR](./blinky-better.md#sharing-data-with-an-isr), and it is why `g_ticks` was declared `volatile` in the first place. Is it safe? Yup.
>
> On RV32 the read of a 32-bit `g_ticks` is a single instruction, so the button ISR can never catch it half-updated, and each variable here has exactly one writer (`g_ticks` from the timer, `last_press_tick` from the button), so there is no read-modify-write race between them. The atomicity rule from that section is satisfied, not bypassed.

## Exercising it in Renode

You don't need a physical board to try this exercise. The bundled Renode platform models PORTA as a real pin-interrupt controller (`support/renode/NXP_PORT.cs`).

Build and launch the simulation:
```sh
make vegaconsole-irq
make sim-vegaconsole-irq
```

The button is GPIOA pin 0. Inputs idle high in the model, mirroring the pull-up, so a press is a falling edge and a release is a rising edge. Drive it from the monitor:

```
(Vegaboard-RI5CY) sysbus.porta OnGPIO 0 false   # press   (falling edge)
(Vegaboard-RI5CY) sysbus.porta OnGPIO 0 true    # release (rising edge)
```

The first press should freeze the heartbeat; press again and it resumes. If you can't watch the LED itself, read its register: the blue LED is bit 22 of GPIOA's `PDOR` at `0x48020000`. Sample it a few times:

```
(Vegaboard-RI5CY) sysbus ReadDoubleWord 0x48020000
```

While the heartbeat runs, bit 22 (`0x00400000`) flips between samples; once you've paused it, the value holds steady. If you want to watch the ISR do its work, turn on access logging for the port before pressing and you'll see the write that clears `ISFR` each time your handler runs:

```
(Vegaboard-RI5CY) sysbus LogPeripheralAccess porta true
```

## Hints

- Don't forget to clear interrupt flags! If the handler fires once and then never again, or fires constantly and the chip hangs, you have missed a clear somewhere.
- If the heartbeat never pauses at all, check the easy things first: is `PORTA_IRQn` enabled at the EVENT_UNIT, and did you add the `case` to the dispatch switch? A press that produces no effect may mean the interrupt never reached your code.
- A 20 ms debounce should be a reasonable starting point for the on-board tactile button. Too short and bounces slip through; too long and a fast double-press registers as one. You may need to adjust up or down if you see unexpected behaviour still.
