# The trap model

Before we can set up and handle *exceptions* or peripheral *interrupts*, we have to teach the CPU what to *do* when one fires. This mechanism for what to *do* will be general-purpose, meaning the same code path for a timer expiring, a button press, a divide-by-zero, or an illegal instruction will be the same. However, before that, let's get our terminology straight by looking at the RISC-V unprivileged ISA specification. If you open section [1.1.6](https://docs.riscv.org/reference/isa/unpriv/intro.html#trap-defn) of the spec, you'll find these definitions:

* *Exception*: "an unusual condition occurring at run time associated with an instruction in the current RISC-V hart [CPU]".
* *Interrupt*: "an external asynchronous event that may cause a RISC-V hart [CPU] to experience an unexpected transfer of control".

The occurrence of any of the above events results in a **trap**, causing the CPU to stop normal instruction flow and transfer control to a *trap handler* so they may be handled with one consistent set of CSRs and one return instruction (`mret`).

By the end of this section, we will be able to deliberately execute an illegal instruction, catch it, print where it happened, and resume the program gracefully.

This section pairs with the `apps/vegaconsole-irq/` example app in the accompanying `vega-quickstart` repository. The trap path we build here lives in `trap_entry.S` (the vector table and assembly stub) and `trap.c` (the C dispatcher), and the `crash` command we use to trigger a trap is in `main.c`. Feel free to open them up and refer to them as you read along.

## What happens during a trap?

When the CPU is happily executing your code and something interrupts it, there are three things the CPU needs to do:

1. Remember where it was, so it can come back.
2. Run a piece of code that figures out *what* happened and handle it in an appropriate way.
3. Resume what it was doing, ideally as if nothing had happened.

RISC-V uses the same machinery for an external interrupt (something outside the core, like a GPIO) and an internal exception (something the core itself noticed, like an illegal instruction). The hardware notifies you of which by setting a single bit.

## The CSRs involved

A handful of CSRs help do the bookkeeping:

| CSR        | What it holds |
|------------|---|
| `mtvec`    | The address the CPU jumps to on any trap and the handling mode. You set this once at boot. |
| `mepc`     | The PC of the instruction that was interrupted. `mret` resumes from here. |
| `mcause`   | Why the trap happened. Top bit = interrupt vs exception; low bits = the specific code. |
| `mtval`    | Extra context (e.g., the faulting address for a memory error). Not always populated. |
| `mstatus`  | A grab-bag of mode bits; the ones we care about are `MIE` (global interrupt enable) and `MPIE` (its saved value, restored on `mret`). |
| `mscratch` | A scratch register the handler can use however it wants. Conventionally a pointer to per-CPU state. |

All of these are actually standard RISC-V CSRs, and unlike `mcycle` from the last section they *are* implemented on the RI5CY core (phew)!

## What the hardware does on trap entry

When a trap fires, the RI5CY CPU does a few things automatically:

1. Copies `pc` (the address of the interrupted instruction) into `mepc`.
2. Copies `mstatus.MIE` into `mstatus.MPIE`, then clears `mstatus.MIE` (effectively ensuring the handler runs with interrupts temporarily disabled so it can't be re-entered by another interrupt).
3. Writes the trap cause into `mcause`.
4. Jumps into the trap vector table whose base address is in `mtvec` (more on that table below).

It's also important to notice what it does *not* do: it does not save any general-purpose registers, and it does not switch stacks. That work is the handler's job (as we'll see below).

On `mret`, the CPU does the reverse: restores `mstatus.MIE` from `mstatus.MPIE` (enabling interrupts again) and jumps back to `mepc`.

## `mtvec`: where the CPU lands

As noted, the last thing the hardware does on a trap is jump into the table that `mtvec` points at. On the RI5CY core, `mtvec` is the base of a hardware *vector table*: rather than landing on one shared entry point, the core jumps to `mtvec_base + offset`, where the offset is fixed by what trapped:

| Offset        | Trap |
|---------------|------|
| `0x00`–`0x7C` | interrupt lines 0–31 (line *N* lands at `N*4`) |
| `0x80`        | reset |
| `0x84`        | illegal instruction |
| `0x88`        | `ecall` |
| `0x8C`        | load/store (LSU) error |

These offsets match the `.vectors` table in the SDK's `startup_RV32M1_ri5cy.S`. So `mtvec` has to point at a real table of instructions, one slot per cause, not at a single function. You set the base once at startup:

```c
extern void trap_entry(void);   /* the vector table, defined in asm */

void install_trap_handler(void) {
    __asm__ volatile ("csrw mtvec, %0" :: "r"((uintptr_t)trap_entry));
}
```

Two constraints come with this. RI5CY masks the low bits of the base, so the table has to be 256-byte aligned. And because each cause is reached at a fixed offset, the slots have to stay a fixed size apart. We'll handle both of these constraints in the codified table further below.

> [!NOTE]
> Generic RISC-V lets `mtvec` choose between *direct* mode (every trap goes to one entry, low bits `0b00`) and *vectored* mode (per-cause entries, low bits `0b01`). RI5CY ignores that field and always vectors, so we have no option but to go through the more complex route of building a table. You will likely have the proper choice on more modern RISC-V MCUs.

## The vector table and trap stub

Below, `trap_entry` is the table. We keep it very simple, having each slot do nothing but jump to a single shared body, and we let the C function dispatcher defined later work out which cause actually fired by reading `mcause`. Note that you *could* give each cause its own handler in the table instead.

```asm
    .section .text.trap_entry, "ax"
    .align   8            # 256-byte aligned base: RI5CY masks the low bits
    .option  norvc        # keep every `j` a full 4 bytes so slots stay 4 apart
    .global  trap_entry
trap_entry:
    .rept 32
    j   trap_body         # 0x00..0x7C: interrupt lines 0..31
    .endr
    j   trap_body         # 0x80: reset
    j   trap_body         # 0x84: illegal instruction
    j   trap_body         # 0x88: ecall
    j   trap_body         # 0x8C: LSU error
    .option  rvc
```

`.align 8` gives the base the 256-byte alignment it needs, and `.option norvc` stops the assembler from compressing any `j` into a 2-byte `c.j`, which would shift every later slot off its offset.

> [!NOTE]
> A few of the lines above are GNU assembler directives rather than instructions. `.rept 32` / `.endr` repeats the enclosed line 32 times so we don't hand-write 32 identical jumps, and `.option norvc` / `.option rvc` toggle the RISC-V compressed-instruction extension off and back on around the table. `.option` and `.align` behave in RISC-V-specific ways, documented in the [binutils RISC-V directives reference](https://sourceware.org/binutils/docs/as/RISC_002dV_002dDirectives.html).

Everything lands in `trap_body`, and that **cannot** be a regular C function. When the trap fires we are running on the user code's stack with the user's register contents. If we just call into C, the compiler will happily clobber any caller-saved register it likes, and when we `mret` the interrupted code will find half its register file scrambled. Things will not go well.

The fix is a short assembly stub: save the caller-saved registers to the stack, call a normal C function to do the real work, restore the registers, and `mret`. Here it is in full:

```asm
trap_body:
    addi    sp, sp, -64       # room for 16 words

    sw      ra,  0(sp)        # save caller-saved regs
    sw      t0,  4(sp)
    sw      t1,  8(sp)
    sw      t2, 12(sp)
    sw      a0, 16(sp)
    sw      a1, 20(sp)
    sw      a2, 24(sp)
    sw      a3, 28(sp)
    sw      a4, 32(sp)
    sw      a5, 36(sp)
    sw      a6, 40(sp)
    sw      a7, 44(sp)
    sw      t3, 48(sp)
    sw      t4, 52(sp)
    sw      t5, 56(sp)
    sw      t6, 60(sp)

    jal     trap_handler      # call the C dispatcher

    lw      ra,  0(sp)        # restore
    lw      t0,  4(sp)
    lw      t1,  8(sp)
    lw      t2, 12(sp)
    lw      a0, 16(sp)
    lw      a1, 20(sp)
    lw      a2, 24(sp)
    lw      a3, 28(sp)
    lw      a4, 32(sp)
    lw      a5, 36(sp)
    lw      a6, 40(sp)
    lw      a7, 44(sp)
    lw      t3, 48(sp)
    lw      t4, 52(sp)
    lw      t5, 56(sp)
    lw      t6, 60(sp)

    addi    sp, sp, 64
    mret
```

It's long...but it only does three things: push 16 registers, call C, pop 16 registers, return. The table above and this body together are the whole `trap_entry.S`.

> [!NOTE]
> **Why those 16 registers and not all 32?**
> The RISC-V calling convention splits registers into *caller-saved* (the function calling you assumes you might trash them) and *callee-saved* (the function you call must preserve them). When we `jal trap_handler`, the C compiler will automatically save and restore the callee-saved regs (`s0`–`s11`, plus `sp` itself) for us. It will *not* save the caller-saved ones, because from its point of view `trap_handler` is the caller. So those are the ones we need to save by hand.

The C software-based dispatcher is small:

```c
void trap_handler(void) {
    uint32_t cause, epc;
    __asm__ volatile ("csrr %0, mcause" : "=r"(cause));
    __asm__ volatile ("csrr %0, mepc"   : "=r"(epc));

    if (cause & 0x80000000u) {            // top bit set = interrupt
        irq_dispatch(cause & 0x7fffffffu);
    } else {                              // top bit clear = exception
        exception_handler(cause, epc);
    }
}
```

The top bit of `mcause` is the "interrupt or exception?" flag, and the low bits identify the specific cause (e.g., cause 2 = illegal instruction). We will fill in `irq_dispatch` (interrupt request dispatch) next page once we have real peripherals configured. For now, let's use the exception path to do something.

## Breaking things on purpose

Let's try to convince ourselves that the trap path actually works by trying to deliberately execute something illegal and catch the result. GCC's `unimp` pseudo-instruction expands to a 4-byte word the chip is guaranteed to reject; it traps with `mcause = 2` (illegal instruction). Add a command to VegaConsole:

```c
static int cmd_crash(int argc, char **argv) {
    PRINTF("about to do something illegal...\r\n");
    __asm__ volatile ("unimp");
    PRINTF("...and back!\r\n");
    return 0;
}
```

Without a trap handler installed, this hangs the chip.

Let's add a handler for the exception path that gives some feedback:
```c
static void exception_handler(uint32_t cause, uint32_t epc) {
    PRINTF("trap! cause=%u epc=0x%08x\r\n", cause, epc);

    /* Skip past the offending instruction so we can return cleanly. */
    __asm__ volatile ("csrw mepc, %0" :: "r"(epc + 4));
}
```

Now a `crash` command will print something like:

```
vega> crash
about to do something illegal...
trap! cause=2 epc=0x000084d4
...and back!
```

The handler ran, told us what happened, nudged `mepc` past the offending instruction, and `mret` resumed the program at the next line.

> [!NOTE]
> **Why `+4` and not `+2`?**
> `unimp` is a 4-byte instruction, so the next one starts 4 bytes later. If the trap had fired on a 2-byte compressed (C extension) instruction, the right increment would be `+2`. A proper handler reads the first 16 bits at `mepc` and checks the low two bits (`0b11` = 4-byte, anything else = 2-byte) to decide, which is what the actual `trap.c` from the quickstart template code does.

## TLDR

- A "trap" is more-or-less an umbrella term in RISC-V for both interrupts and exceptions. They share one return (`mret`) and one set of CSRs, and on RI5CY we funnel every cause into one shared handler body.
- On trap entry, the hardware saves the PC to `mepc`, stashes the interrupt-enable bit, writes the cause to `mcause`, and vectors into the table at `mtvec`.
- On RI5CY, `mtvec` is the base of a hardware vector table: each cause lands at `mtvec_base + offset`, the base must be 256-byte aligned, and the slots must stay 4 bytes apart. Every slot jumps to one shared asm stub that saves the caller-saved registers and calls a C dispatcher.
- The top bit of `mcause` distinguishes interrupts from exceptions. The low bits identify the specific cause.
- You can recover from an exception by advancing `mepc` past the offending instruction and `mret`-ing.
- The CSRs and `mret` here are standard RISC-V; the vector table is RI5CY-specific. The chip-specific glue code (i.e., to decide which peripheral triggered an interrupt? and which handler do we run?) comes next.
