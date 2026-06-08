# UART

This section pairs with the `apps/hello-uart/` example app in the accompanying `vega-quickstart` repository, which is a small UART program to help understand the basics. Feel free to open it up and refer to it as you read along.

UART stands for Universal Asynchronous Receiver-Transmitter, which is a long way of saying "two wires, no clock". One wire (TX) carries bytes from the chip to whoever's listening, the other (RX) brings bytes the other way. Both sides must agree ahead of time on how fast they'll talk: the *baud rate* (which we briefly mentioned in the first section, using `115200` in our case). There is no shared clock, no acknowledgement, no framing beyond a single start bit and a single stop bit. It is probably the simplest serial protocol there is, which is why it has been showing up on microcontrollers since forever and seems to show no sign of leaving.

## What's on the wire?

When the communication line is idle, TX sits high. To send a byte, the transmitter pulls TX low for one bit period; that's the **start bit**, and it's how the receiver knows a byte is coming. Then it shifts out eight data bits (one byte), least significant bit first, each held for one bit period. Finally it lets the line go high again for at least one bit period; that's the **stop bit**. After that the line stays high until the next byte.

> [!NOTE]
> UART is **little-endian at the bit level**: within each byte, the least significant bit (`b0`) goes out first and the most significant (`b7`) last. So `'Y'` (`0b01011001`) appears on the wire as the sequence `1, 0, 0, 1, 1, 0, 1, 0`, which is the binary value read right-to-left. Endianness here is about bit order inside one byte; UART has no notion of multi-byte word order, so concepts like big- vs little-endian integers are a layer up, decided by whatever protocol you build on top.

```
idle start  b0   b1   b2   b3   b4   b5   b6   b7  stop idle
─────┐    ┌────┐         ┌─────────┐    ┌────┐    ┌─────────
     │    │    │         │         │    │    │    │
     └────┘    └─────────┘         └────┘    └────┘
     <--->
  1 bit period

byte = 0b01011001 = 0x59 = 'Y'; sent LSB-first (little-endian) on the wire
```

> [!NOTE]
> `0b01011001` is the [ASCII](https://en.wikipedia.org/wiki/ASCII) encoding of the uppercase letter `Y`. ASCII assigns each printable character a 7-bit code (the high bit is zero in 8N1), so when a UART carries text, each byte on the wire is just the ASCII value of one character.

That's the whole protocol. There is no length field, no checksum, no addressing, nothing. Both ends just have to count carefully. If they disagree about the bit period by more than a few percent, the receiver samples in the wrong place and you get garbage.

The bit period comes from the baud rate, which is measured in symbols per second. Because each UART symbol carries exactly one bit, 115200 baud means 115200 bits/s, or about 11520 bytes/s on the wire (one byte takes ten bit times: one start + eight data + one stop). 115200 is the most common choice for microcontroller debug consoles, and it's what we used in the blinky debug console.

> [!NOTE]
> The "8N1" you'll see written on serial-terminal config screens means **8** data bits, **N**o parity bit, **1** stop bit. That's what we use here, and what almost every embedded UART defaults to. Other framings exist (7E1, 8O2, etc.), but you will likely rarely see them.

## LPUART0 on the RV32M1

The RV32M1 has several (Low-Power) UART peripherals. We've already been using LPUART0 the whole time, but just glossed over it until now: `BOARD_InitDebugConsole` in `apps/blinky/board.c` configures it at 115200 baud and hands it to the SDK's debug console. That's why every `PRINTF` from blinky magically lands on the J12 USB serial port!

Like every peripheral on this chip, LPUART0 is a small block of memory-mapped registers. From `rv32m1-sdk/devices/RV32M1/RV32M1_ri5cy.h` (or Section 56.3.1 of the reference manual):

```c
#define LPUART0_BASE  (0x40042000u)
#define LPUART0       ((LPUART_Type *)LPUART0_BASE)
```

The four main fields of `LPUART_Type` you'll need to deal with the most are:

| Offset | Name   | What it's for                                                |
|--------|--------|--------------------------------------------------------------|
| `0x10` | `BAUD` | Clock divider that sets the baud rate                        |
| `0x14` | `STAT` | Status flags: TX buffer empty, RX buffer full, errors, ...   |
| `0x18` | `CTRL` | Enables for transmit, receive, parity, interrupts, ...       |
| `0x1C` | `DATA` | The TX/RX shift register; write a byte to send, read to receive |

Almost everything we do with the UART is one of two things: writing a byte to `DATA` or reading a byte from `DATA`. Everything else is one-time setup.

### How is baud calculated?

Feel free to skip this section if math spooks you, as we'll be taking a brief look at the `BAUD` register and the basic math that must be done to derive correct settings. Note, the configuration discussed below is specific to the peripherals of our MCU, but other MCU UART peripherals typically require similar calculations to derive a correct baud rate, so it's super useful to know this stuff!

In our case, inside the LPUART, the input clock is divided by an oversampling factor and then by a sub-baud-rate divider (**SBR**) to produce the bit clock. The oversampling factor is set by the **OSR** field, which holds the factor *minus one* (it resets to 15, i.e. oversample by 16), which is why the formula below uses `OSR + 1`. With LPUART0 sourced from FIRC (Fast Internal Reference Clock) at 48 MHz:

```
baud = source_clock / ((OSR + 1) × SBR)
```

For 115200 baud, OSR = 15 and SBR = 26 give:

```
48_000_000 / ((15 + 1) × 26) = 48_000_000 / 416 ≈ 115384
```

That's about 0.16% off the nominal 115200, well within the ~3% of tolerance UART needs. The SDK's `LPUART_SetBaudRate` does the calculation for you, and now you can look at it and know it's not magic.

> [!NOTE]
> In general, if you ever inherit a board where the serial console "almost works" but drops things randomly, the first thing you might want to check is how the UART registers and the above calculations are done for it. Mismatched clock trim or a clock tree configured for the wrong source can push the error past UART's tolerance and produce all kinds of spooky behaviour.

## Tracing a single byte

Sending a byte ultimately comes down to one inlined helper in `rv32m1-sdk/devices/RV32M1/drivers/fsl_lpuart.h`:

```c
static inline void LPUART_WriteByte(LPUART_Type *base, uint8_t data) {
    base->DATA = data;
}
```

That's it: a single 32-bit store to `0x4004201C` (LPUART0 base + the `DATA` offset). The hardware latches the byte into its TX shift register and clocks it out PTC8 at the configured baud rate. The catch is that you can only write `DATA` once the previous byte has cleared, otherwise you stomp on it. So `hello-uart` wraps the store in a loop against the TX-empty flag:

```c
static void uart_putc(char c) {
    while (!(LPUART_GetStatusFlags(BOARD_DEBUG_UART) & kLPUART_TxDataRegEmptyFlag))
        ;
    LPUART_WriteByte(BOARD_DEBUG_UART, (uint8_t)c);
}
```

`LPUART_GetStatusFlags` is one masked read of `STAT` field.

To receive, you do essentially a mirror of the same thing above: spin until `LPUART_RxDataRegFullFlag` is set, then read `DATA`.

To build the example and run it, run the below:
```sh
make hello-uart
make sim-hello-uart   # or make flash-hello-uart on real hardware
```

On real hardware, type some characters and they echo back capitalized; press Enter for a fresh prompt. In simulation, the Renode monitor drives the UART for you; at the Renode prompt, run

```
lpuart0 WriteLine "hello" True
```

to push the string `hello` into LPUART0's RX as if you'd typed it (the trailing `True` appends a carriage return, which the firmware treats as Enter). The capitalized echo shows up on the same UART analyzer window.

> [!NOTE]
> **How did `PRINTF` statements in previous examples work?**
>
> In the previous blinky program, we had `PRINTF` calls printing out to this same UART interface.
> This is because that code called `BOARD_InitDebugConsole`, which configured LPUART0 and hooked the SDK's debug-console module to it. `PRINTF(...)` expands to `DbgConsole_Printf(...)`, which eventually calls `LPUART_WriteByte` per character. Pretty much the same thing we are doing in `hello-uart`, but with a nice wrapper around it.

## RISC-V aside: CSRs and the cycle counter

Great, now we know UART! Let's take a sidestep to introduce something RISC-V specific that we can use to deepen our knowledge and which we'll use again in the next section: **Control and Status Registers**, or CSRs. CSRs are a small bank of architectural registers that aren't part of the integer register file; you access them with their own family of instructions (`csrr`, `csrw`, `csrrs`, `csrrc`). The CSR we'll focus on now is a free-running cycle counter that increments once per clock; on a standard RV32 core it's called `mcycle` (CSR `0xB00`), and reading it tells you how many cycles have passed.

> [!NOTE]
> Unlike the `uart_putc`/`uart_getc` code above, the helpers in this aside are *not* part of `hello-uart`; they're standalone teaching snippets you can drop into any app's `main.c`. We'll reuse them in this chapter's challenge and again in the next chapter. None of this is needed to use the UART, it's purely a RISC-V detour into how the chip exposes its own state.

> [!WARNING]
> This section will not work in Renode, you'll need a physical board. We don't have support in Renode for the RI5CY PULP CSR performance counters, so the `csrr`/`csrw` instructions below trap as illegal instructions and the program faults/crashes. If you're following along in the simulator, feel free to read through this section but skip running it.

There's one small wrinkle on the RV32M1's RI5CY core: it doesn't actually implement the standard `mcycle` CSR. Reading `mcycle` just returns zero. Instead, RI5CY exposes its cycle count through a PULP-specific performance counter, `pccr0`, at CSR `0x780`, and that counter is disabled at reset. So before the first read we need to turn it on by writing the PULP enable CSRs `pcer` (`0x7A0`, per-event enable mask) and `pcmr` (`0x7A1`, global enable):

```c
static inline void enable_perf_counters(void)
{
    // PCER bit 0 = count cycles
    __asm__ volatile("csrw 0x7A0, %0" ::"r"(0x1));
    // CMR bit 0 = global enable; leave bit 1 (saturate) clear so the
    // counter wraps and unsigned subtraction in (b - a) keeps working.
    __asm__ volatile("csrw 0x7A1, %0" ::"r"(0x1));
}
```

Call `enable_perf_counters()` once early in `main`. Then we can read the counter with one instruction of *inline assembly*:

```c
static inline uint32_t csr_cycles(void) {
    uint32_t v;
    __asm__ volatile ("csrr %0, 0x780" : "=r"(v)); /* pccr0 */
    return v;
}
```

> [!NOTE]
> RV32 CSRs are 32 bits wide, so `pccr0` (and `mcycle` on a standard core) wraps. At 48 MHz that's every `2^32 / 48e6 ≈ 89 seconds`. The things we'll measure (a UART byte, a few function calls) finish in microseconds, so we just read the low 32 bits and let unsigned arithmetic handle a wrap in the subtraction `b - a` below.

The `__asm__ volatile (...)` construct is GCC's *inline assembly*, a way to drop hand-written instructions into a C function while still letting the compiler manage register allocation around them. The string `"csrr %0, 0x780"` is the instruction template; `%0` gets substituted with whatever register the compiler picks for the output operand `v`, and the `"=r"(v)` constraint is what tells it to allocate one general-purpose register and write the result of the instruction into `v`. `volatile` keeps the compiler from doing any screwy optimization (e.g., caching or reordering), which is important because two back-to-back reads of the cycle counter have different values. So `csrr rd, csr` is a real RISC-V instruction; we are not calling a helper function here, the compiler emits exactly this one CSR-read instruction. Now we can ask the chip how many cycles a particular operation takes:

```c
uint32_t a = csr_cycles();
uart_putc('A');
while (!(LPUART_GetStatusFlags(BOARD_DEBUG_UART) & kLPUART_TransmissionCompleteFlag))
    ; /* wait for the shift register to fully drain */
uint32_t b = csr_cycles();
PRINTF("one byte took %u cycles\r\n", (unsigned)(b - a));
```

Run the math against expectations. At 48 MHz and 115200 baud, one byte on the wire takes 10 bit-periods × (48 MHz / 115200) ≈ 4170 cycles, and that's what the snippet above prints. There's a subtlety worth knowing though. Our `uart_putc` only waits on `kLPUART_TxDataRegEmptyFlag` (TDRE), which goes high when the data register can accept the next byte (the hardware has shoved the previous one into the TX shift register), not when it has finished clocking out. If we read the cycle counter the moment `uart_putc` returns, we just time the store into `DATA` and miss the thousands of cycles the hardware then spends shifting the bits onto the wire; you'd see something like 90 cycles. So before the second read we also wait on `kLPUART_TransmissionCompleteFlag` (TC), which clears only once the shift register is fully empty. Regardless, you've just measured a real piece of timing on a real chip with one CSR read, neat! These are the kinds of thing CSRs are *for*!

## TLDR

* UART is two wires (TX, RX), no clock, and requires an agreed-upon baud rate. Each byte is a start bit, eight data bits LSB-first, and a stop bit.
* LPUART0 on the RV32M1 is a memory-mapped peripheral with a handful of registers to configure. Four registers worth knowing are: `BAUD`, `CTRL`, `STAT`, `DATA`.
  - Sending a byte is one **write** to `DATA`.
  - Receiving a byte is one **read** of `DATA`.
  - The TX-empty and RX-full bits in `STAT` tell you when a send/receive is allowed.
* `PRINTF` go to LPUART0 because `BOARD_InitDebugConsole` sets up the SDK's debug-console module to call `LPUART_WriteByte` underneath. No magic beyond that.
* A cycle counter is a CSR you can read with one inline-assembly instruction. On RI5CY the standard `mcycle` isn't supported, so we read PULP's `pccr0` (CSR `0x780`) after enabling it through `pcer` and `pcmr`.
