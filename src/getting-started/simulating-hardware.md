# Simulating hardware

Real hardware is great...but sometimes slow to iterate on and not readily accessible. To talk with real hardware, you have to rebuild your firmware, flash it over a debug probe, and then poke at the board (blinking an LED, squinting at a serial terminal, etc) to see whether anything is working. A simulator sidesteps all of this. It runs your firmware on a virtual copy of the board, entirely inside your computer, and gives you direct visibility into what the code is doing. Most importantly, you can also simulate a board you don't physically own!

This training uses [Renode](https://renode.io/), an open-source simulator from Antmicro that can model full embedded systems (CPU, memory, and peripherals). Renode already ships with a basic platform description for the VEGAboard, so we can run firmware on it out of the box.

> [!NOTE]
> It turns out the platform description provided in the official Renode repository is not very complete.
> The quickstart repository has custom definitions under `support/renode`.
> We won't go into detail about these files, but feel free read to read through them if interested in learning more about Renode.

## Renode basics

Before we run anything, it helps to know three Renode concepts:

1. A *platform description* (`.repl` file) lists the virtual hardware: what CPU, how much memory, which peripherals live at which addresses. Renode includes a *basic* one for the VEGAboard at [platforms/boards/vegaboard_ri5cy.repl](https://github.com/renode/renode/blob/master/platforms/boards/vegaboard_ri5cy.repl).
    - The quickstart repository for these labs contains a more advanced one under `support/renode`
2. A *Renode script* (`.resc` file) is a small recipe that builds a machine from a `.repl`, loads firmware into it, and wires up things like UART windows. Renode also includes [scripts/single-node/vegaboard_ri5cy.resc](https://github.com/renode/renode/blob/master/scripts/single-node/vegaboard_ri5cy.resc) that does this for our board.
3. The *monitor* is Renode's interactive prompt. When you start Renode you land in the monitor and type commands such as `start`, `pause`, or `showAnalyzer` to drive the simulation.

You don't need to write any of this from scratch for the VEGAboard. The quickstart repository does all the setup for. You'll primarily just be running a handful of monitor commands to explore the system as it runs.

## Running blinky in Renode

Let's jump right into it! We'll be simulating the same blinky app we developed in previous sections.

First, make sure blinky has been built. From the repository root, inside the dev container, run:
```sh
make blinky
```

This produces the ELF file Renode will load:
```
build/blinky/blinky.elf
```

From the same shell inside the container, launch Renode:

```sh
renode
```

> [!NOTE]
> If you see a line like `Couldn't start UI - falling back to console mode`, that's fine.
> It just means Renode didn't find a graphical display (typical when you're inside a container over SSH or a remote connection).
> The monitor prompt works identically either way.

At the `(monitor)` prompt, point Renode at your ELF file and include the bundled VEGAboard script:

```
(monitor) $bin=@/workspaces/vega-quickstart/build/blinky/blinky.elf
(monitor) include @/workspaces/vega-quickstart/support/renode/vegaboard_ri5cy.resc
```

The first line sets a variable named `$bin` that the platform Renode script reads to decide which binary to load. The second line executes the Renode script, which creates the virtual machine, wires up the CPU and peripherals, loads your firmware into flash, and calls the monitor command `showAnalyzer lpuart0` for you so the simulated UART is attached to your terminal from the start. After it finishes you'll see a new prompt that reflects the name of the machine:

```
(Vegaboard-RI5CY)
```

Now start the emulation. `start` and its one-letter alias `s` both work:

```
(Vegaboard-RI5CY) start
```

You should see the UART output from `main.c` appear on the terminal:

```
lpuart0: RV32M1-VEGA RI5CY baremetal app
lpuart0: Starting to blink LED...
```

That's the `PRINTF()` calls in `main.c` writing to `lpuart0`, which the bundled script routed to your terminal. If you don't see them, double-check the ELF path in `$bin`.

To stop cleanly, pause and quit the monitor:

```
(Vegaboard-RI5CY) pause
(Vegaboard-RI5CY) quit
```

## Watching the LED

The quickstart's platform at `support/renode/vegaboard_ri5cy_platform.repl` properly defines GPIO ports and the logic backing them (via the `NXP_GPIO` peripheral in `support/renode/NXP_GPIO.cs`).

Thus, writes land in a real register inside Renode and the LED's output state is observable from the monitor.

As discussed in earlier sections, the LED is on GPIOA pin 24 (see `BOARD_LED_GPIO` and `BOARD_LED_GPIO_PIN` in `board.h`). GPIOA lives at `0x48020000`; the PDOR register (the current output latch) is at offset `0x00`.

While the simulation is running, sample it from the monitor:
```
(Vegaboard-RI5CY) sysbus ReadDoubleWord 0x48020000
```

Run it a few times. The value alternates between `0x00000000` and `0x01000000` as the firmware toggles pin 24. Bit 24 set is the LED on; clear is off.

For a live view of every access `gpioa` receives, turn on global peripheral access logging before `start` (or pause first):

```
(Vegaboard-RI5CY) sysbus LogAllPeripheralsAccess true
```

Each iteration of the `while (1)` loop in `main.c` then prints a line showing a write of `0x01000000` to offset `0x0C` on `gpioa` (PTOR, the toggle register). The rate at which those writes appear is your blink rate. Pause the simulation, change the iteration count in `delay()`, rebuild, and watch the log speed up or slow down. The logging is noisy but useful while you're learning, since every read and write the CPU makes shows up with its address, value, and the address of the instruction that issued it.

## A faster workflow

Doing all of the commands above will quickly become tedious.
Luckily for you, we've defined a helper in the base `Makefile` that automates
launching Renode and loading the specified application in a single command.

To launch and simulate the blinky application, simply run:
```sh
make sim-blinky
```

This will take you straight from a shell prompt to a running simulation of blinky.
Going forward, any application we write can be simulated by simply running `make sim-<app>`!
