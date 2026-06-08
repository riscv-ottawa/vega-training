# Building VegaConsole

Now that we know the basics of UART, let's build something cool: a read-eval-print loop (REPL) *command interpreter*. This interpreter will read a line from the user, determine if they passed a valid pre-defined command, then run the corresponding code for the given command and print its result. That's all a shell really is. Let's build it!

The full source lives at [`apps/vegaconsole/main.c`](https://github.com/riscv-ottawa/vega-quickstart/blob/main/apps/vegaconsole/main.c) in the accompanying [`vega-quickstart`](https://github.com/riscv-ottawa/vega-quickstart) repository.

## Getting loopy

Every line-oriented REPL ever written has the same skeleton:

```
1. print a prompt
2. read characters one at a time, into a buffer
3. when the user presses Enter, terminate the buffer with '\0'
4. split it into argv[0..n-1]
5. look argv[0] up in a command table; call its handler
6. go to 1
```

That is exactly what `main.c` does. The interesting part is what commands we decide to support (the options are limitless) and what actually happens when the user enters one of these commands.

## The command table

Commands are defined in a single table that the rest of the code automatically searches and supports. Each entry is a triple of name, one-line help, and function pointer:

```c
typedef int (*cmd_fn)(int argc, char **argv);

typedef struct {
    const char *name;
    const char *help;
    cmd_fn      run;
} command_t;

const command_t commands[] = {
    { "help",   "list available commands",                cmd_help   },
    { "led",    "led <color> <on|off>",                   cmd_led    },
    { "echo",   "echo <text...>",                         cmd_echo   },
};
```

The classic `argc` (argument count) and `argv` (argument array of strings) makes the command bodies look like miniature `main` functions. If you've written a Unix CLI tool before, you've likely seen this exact dispatch pattern.

`help` walks the table and prints each entry. `led` looks the color up in a small `{name, pin}` array and calls `GPIO_WritePinOutput`. `echo` glues `argv[1..]` back together with spaces.

## Line buffering and the editor

Reading characters one at a time means we get to decide what counts as "a line". The minimum viable rule is "everything until the user presses Enter", but a serial terminal sends a few control bytes that are worth handling explicitly. Here's the loop, lightly trimmed:

```c
char  line[LINE_MAX];
int   len = 0;
PRINTF(PROMPT);

for (;;) {
    int c = GETCHAR();

    if (c == '\r' || c == '\n') {
        PRINTF("\r\n");
        line[len] = '\0';
        dispatch(line);
        len = 0;
        PRINTF(PROMPT);
    } else if ((c == 0x7f || c == 0x08) && len > 0) {  /* DEL or BS */
        --len;
        PRINTF("\b \b");
    } else if (c >= 0x20 && c < 0x7f && len < LINE_MAX - 1) {
        line[len++] = (char)c;
        PUTCHAR(c);
    }
    /* anything else is silently dropped */
}
```

The three branches:

1. **Enter** (`\r` from most terminals): null-terminate, hand the buffer to `dispatch`, reset, print a fresh prompt.
2. **Backspace** (DEL `0x7f` or BS `0x08`, depending on the terminal): drop the last character if there is one, then emit the `\b \b` sequence to make the user's screen agree with us.
3. **Anything printable**: append to the buffer and echo. Without that echo, the user types into the void.

`PUTCHAR` and `GETCHAR` are macros from `fsl_debug_console.h` that resolve to `DbgConsole_Putchar` and `DbgConsole_Getchar`. They reach the same `LPUART_WriteByte` / `LPUART_ReadByte` we traced last page.

> [!NOTE]
> **What your terminal actually sends.**
> Pressing Enter on most modern terminals sends a single `\r` (`0x0D`); a few send `\r\n`. Backspace varies even more: macOS Terminal, modern Linux terminals, and minicom send DEL (`0x7f`) by default; some older or stricter terminals send BS (`0x08`). The `0x20`-`0x7e` printable range covers everything you'd expect.
>
> Usually, if you press any **arrow key**, you'll see things like `^[[A` (e.g., if you press up); this is because they send a escape sequence of bytes. The loop we made actually has one additional branch to drop every byte in these escape sequences. Real shells parse those sequences to give you history and cursor movement, we keep it simple and skip that in our implementation.

## Tokenizing without `strtok`

Splitting `"led red on"` into `argv` is the kind of thing you'd reach for `strtok` from the C standard library. Given we don't have a standard library, we do something similar to `strtok` with a tiny custom tokenizer:

```c
static int tokenize(char *line, char **argv, int max) {
    int argc = 0;
    char *p = line;
    while (*p && argc < max) {
        while (*p == ' ' || *p == '\t') *p++ = '\0';
        if (!*p) break;
        argv[argc++] = p;
        while (*p && *p != ' ' && *p != '\t') ++p;
    }
    return argc;
}
```

It iterates over the line buffer, replacing runs of whitespace with `'\0'` and pointing each `argv[i]` at the start of a token. After it returns, the buffer looks like `"led\0red\0on"` and `argv` points into the right places. The great thing about this is that we don't do any allocations or copies!

`dispatch` then linear-searches the command table for `argv[0]` and calls the handler. With such few commands a linear search is obviously fine, but if we were to grow this command table a lot...we might want to reach for a hash or sorted lookup data structure instead.

## Try it

Build and run, either path:

```sh
make sim-vegaconsole       # in Renode
make flash-vegaconsole     # to a real board, then `make serial`
```

A session looks like:

```
=== VegaConsole ===
type 'help' to see what's available.
vega> help
commands:
  help     list available commands
  led      led <color> <on|off>
  echo     echo <text...>
vega> led blue on
vega> echo hello there
hello there
```

> [!NOTE]
> Don't forget if simulating with Renode, you need to use commands like `lpuart0 WriteLine "led blue on" True` to write to the serial port.

In Renode, the LED state shows up on the simulated GPIOA peripheral; you can confirm it from the monitor by inspecting `PDOR` (`sysbus ReadDoubleWord 0x48020000`). On real hardware, you should see the actual LED change.

## What's wrong with this design?

Great, we've got our fancy REPL command interpreter working...however, a few things are not great about our current implementation:

**The CPU is busy-waiting on a human.** Every `GETCHAR` call spins on the `STAT` register until you press a key. While it spins, `main` cannot do anything else. There is no option to have something like "meanwhile, blink an LED in the background" or "meanwhile, sample a sensor every millisecond". The simplest way to see that this is true is to add a command that continually does something in a loop. For example, if we added a `blink <hz>` command that toggles the LED in a loop with `delay()`: while it is running, the REPL will be frozen until it returns. The worst part is that the CPU isn't even *doing* anything during the spin in `GETCHAR`; it's just rejecting the same `STAT` bit (tens of millions of times a second)!

**The whole loop is *foreground* work.** Even if we wanted background work, there is no mechanism for it. Every line of code we've written since blinky has been on the main thread, scheduled by `main` and nothing else. That's fine for a simple blinky and it's kinda fine for our REPL, but the moment we want something more, like a heartbeat tick *and* a REPL *and* a button reaction, we are doomed.

<!--and noticing them is the bridge to the next section.-->
The fix to both problems is the same mechanism: **interrupts**. The next section is about defining work and teaching the chip to call us back when that work is done or something happens, instead of asking us to keep checking. We'll continue with the VegaConsole code as an example, but now we'll just take the busy-wait out of `GETCHAR` and learn how to let the CPU sleep while it has nothing to do.

## TLDR

- A REPL (read-eval-print loop) is a fixed loop, in our case we: read, tokenize, dispatch, then print.
- A command table of `{name, help, fn}` gives you readable code, easy `help` output, and a single place to register new commands. Linear search is fine until the table gets *big*.
- For line editing on a serial terminal, it's nice/necessary to handle Enter, Backspace (DEL or BS), and ignoring escape sequences for things such as arrow keys.
- `tokenize` rewrites the input buffer in place to produce `argv`. Simple, and no memory allocator required.
- `GETCHAR` is busy-waiting on the UART status register. That works, but it pins `main` in a forever busy-loop and leaves no room for anything else. We are going to fix this in the next section.
