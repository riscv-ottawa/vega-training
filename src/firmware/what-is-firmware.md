# Wait, what is firmware?

In our case, *firmware* is the software that will be running directly on our microcontroller.

> [!NOTE]
> **Wait, what is a *microcontroller*?**
>
> A *microcontroller* unit (MCU) is a whole tiny computer packed onto a single chip: a CPU, a small amount of memory (both flash for code and RAM for data), and a fixed set of peripherals, all sharing the same piece of silicon. The VEGAboard's main chip, the RV32M1, is one example.
>
> Different from the *microprocessor* in your laptop, which only handles the CPU part and relies on separate chips for RAM, storage, and I/O. Because an MCU has everything on-board (and typically has it in *much* smaller proportions), it can be small, cheap, and low-power enough to live inside a thermostat, a car's door lock, a pair of headphones, etc - all of which run one (or a handful of) small dedicated fixed programs (i.e., the firmware).

Firmware is the code that lives in the chip's flash memory, starts running the instant power is applied, and continues executing until power is removed (or the system crashes ;D).
Unlike a desktop application, it sometimes has no operating system underneath it at all.
It is simply a program that talks to hardware.

Because the hardware is so much smaller than a laptop (often a few hundred kilobytes of flash, tens of kilobytes of RAM, and a single CPU running in the tens of megahertz), firmware is written with those constraints in mind. At times, every byte of memory needs to be minimized, every clock cycle accounted for, and the program has to handle *everything* itself: setting up the chip after reset, reacting to signals from the outside world, and keeping track of time.

The sections below walk through the three ideas that set firmware apart from "regular" software: how a program starts when there is no operating system to launch it, how a single CPU juggles many things at once, and how the code actually *interacts* the physical world around it.

## How execution starts

On a desktop, your operating system loads your program into memory, sets up its stack, and calls `main()`. On a bare-metal microcontroller, there is no operating system to do any of that. The chip has to bring itself up from nothing.

When the VEGAboard powers on (or you press reset), the CPU begins executing from a fixed, known address in flash. Typically, the very first thing it finds there is the *vector table*: a small array of addresses that tell the CPU where to jump for important events, with the very first entry being the *reset handler*. The reset handler is just a function, usually written in a mix of assembly and C, and its job is to prepare the chip to run your code.

That preparation does a few things in order:

1. Set up the *stack pointer* so the CPU has somewhere to store local variables and return addresses.
2. Copy any initialized global variables (the `.data` section) from flash into RAM, since RAM starts out with undefined contents.
3. Zero out uninitialized globals (the `.bss` section), so variables declared without an initializer start at `0`.
4. Optionally configure the chip's clocks, caches, and other essentials.
5. Finally, call `main()`.

Only *after* all of the above does your `main()` function actually start running. And unlike on a desktop, `main()` on a microcontroller almost never returns. There is nothing for it to return *to*. Instead it typically ends with an infinite `while (1)` loop that does the real work forever (as you gain experience in this area, you'll learn that busy looping forever is typically a bad idea and that's where things like deep sleep and time-based scheduling comes in).

> [!NOTE]
> If you want to see this process in full detail (for a different chip, but with the same ideas), Memfault's [Zero to main()](https://interrupt.memfault.com/blog/zero-to-main-rust-1) series walks through every step of startup code, from the reset vector to the first line of `main`.

## What is a peripheral?

Doing random computation is great and all, but how can computation on something like the VEGAboard result in sensing or actuation in the real physical world?

A CPU on its own can add numbers and move data around in memory, but it cannot blink an LED, send a byte over a wire, or sample a voltage. Those jobs are handled by *peripherals*: dedicated hardware blocks that sit next to the CPU inside the microcontroller. Typical peripherals include GPIO (general-purpose I/O pins), UART (serial communication), SPI and I2C (for talking to external chips), timers, and ADCs (analog-to-digital converters).

One thing to note about peripherals is that they run independently of the CPU. Once you configure a UART peripheral and hand it a byte to transmit, it shifts the bits out on its own while the CPU goes off to do something else. In that sense, a microcontroller is really a small CPU surrounded by a dozen tiny, single-purpose coprocessors.

The way the CPU talks to these peripherals is called *memory-mapped I/O*. Each peripheral has a block of addresses reserved for it in the chip's address space, and within that block sit a handful of *registers*, each controlling one aspect of the peripheral. Writing to an address directly changes a peripheral's behaviour. Reading an address gives you a peripheral's current state.

For example, the VEGAboard's LED is connected to pin 24 of GPIO port A. The GPIOA peripheral lives at address `0x48020000` and exposes six 32-bit registers back-to-back in memory:

```text
GPIOA @ 0x48020000

0x48020000  ┌───────────────────────────┐
            │ PDOR  (RW)                │  Output latch: 1 bit per pin.
0x48020004  ├───────────────────────────┤
            │ PSOR  (WO)                │  Write 1 to *set* PDOR bits.
0x48020008  ├───────────────────────────┤
            │ PCOR  (WO)                │  Write 1 to *clear* PDOR bits.
0x4802000C  ├───────────────────────────┤
            │ PTOR  (WO)                │  Write 1 to *toggle* PDOR bits.
0x48020010  ├───────────────────────────┤
            │ PDIR  (RO)                │  Reads back the actual pin state.
0x48020014  ├───────────────────────────┤
            │ PDDR  (RW)                │  Direction: 0 = input, 1 = output.
0x48020018  └───────────────────────────┘

("RW" = read+write, "WO" = write-only, "RO" = read-only)
```

Within a single register, each of the 32 bits maps to one pin on the port. For `PDOR`, bit 24 is the one wired to the LED:

```text
PDOR @ 0x48020000
   bit 31        bit 24                                          bit 0
    │             │                                               │
    v             v                                               v
   ┌─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┐
   │ │ │ │ │ │ │ │L│ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │
   └─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘
                      L = LED (1 = on, 0 = off)
```

Setting bit 24 of `PDOR` turns the LED on; clearing it turns the LED off. In C, that looks roughly like:

```c
volatile uint32_t *pdor = (uint32_t *)0x48020000;
*pdor |= (1 << 24);   // LED on
*pdor &= ~(1 << 24);  // LED off
```

You will rarely write code quite *that* raw in practice. Vendor-supplied software development kits (SDKs) wrap these registers in named structs and helper functions so you can write something like `GPIO_PinWrite(GPIOA, 24, 1)` instead. But underneath those abstractions, every peripheral interaction bottoms out in a load or store to a specific memory address.

## How is multitasking done on MCUs?

Firmware will often do stuff blink an LED, read a sensor, respond to a button, and print out data, all "at the same time"...how?

The simplest and most common pattern is a *super loop*: one big `while (1)` inside `main` that checks each task in turn and does a bit of work for each one. It looks something like this:

```c
int main(void) {
    setup_everything();
    while (1) {
        update_led();
        read_sensor_if_ready();
        handle_uart();
    }
}
```

As long as none of the individual tasks block for too long, each one gets serviced often enough to *feel* simultaneous. The blinky application you'll meet in the next section is the most minimal version of this pattern: a single `while (1)` that toggles a GPIO pin and waits.

The super loop breaks down when something needs to happen *right now*, for example, reacting the microsecond a pulse arrives on a pin. For that, microcontrollers provide *interrupts*: hardware signals that pause whatever the CPU is doing, jump to a small handler function to deal with the event, and then resume the interrupted code. We'll dedicate a later section to interrupts and timers, but the short version is that well-designed firmware usually combines both: a super loop doing the slow, steady work, and interrupts handling anything that is time-sensitive.

When the super loop stops scaling (many independent tasks, strict timing deadlines, multiple developers working in parallel), the next step up is a *real-time operating system*, or RTOS. An RTOS lets you write each task as if it had the CPU to itself and takes care of switching between them. The RTOS section of this training covers this briefly by introducing [Zephyr](https://www.zephyrproject.org/).

## TLDR

- Firmware is the software that runs directly on a microcontroller (MCU), typically with no general purpose operating system beneath it and tight limits on memory and CPU speed.
- An MCU starts executing from a fixed address in flash. Startup code sets up the stack, initializes memory, and eventually calls `main()`, which never returns.
- Peripherals are small, independent hardware blocks (GPIO, UART, timers, etc) that the CPU drives by reading and writing specific memory addresses. Every firmware operation eventually boils down to a load or store instruction (defined in the [RV32I spec](https://docs.riscv.org/reference/isa/unpriv/unpriv-index.html)!).
- A single CPU core fakes multitasking through a super loop plus interrupts. When applications become too complex, people typically use an RTOS for better abstraction and task handling.
