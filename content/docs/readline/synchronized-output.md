---
date: '2026-02-23T15:00:00+01:00'
draft: false
title: 'Synchronized Output'
weight: 13
---

Synchronized output (Mode 2026) prevents screen tearing during rapid terminal redraws. When enabled, the terminal buffers all output until the mode is disabled, then paints the entire frame atomically. This is particularly useful for readline — completion lists, multi-line prompts, and screen repaints all cause multiple write calls that can tear without synchronization.

## How It Works

Mode 2026 uses two escape sequences:

| Sequence | Name | Purpose |
|----------|------|---------|
| `ESC[?2026h` | BSU (Begin Synchronized Update) | Start buffering output |
| `ESC[?2026l` | ESU (End Synchronized Update) | Flush buffer and paint |

Everything written between BSU and ESU is held by the terminal and rendered in a single
operation. On terminals that don't recognize Mode 2026, the sequences are silently ignored
— so it's always safe to send them.

## Quick Start

```java
import org.aesh.terminal.tty.TerminalConnection;
import org.aesh.terminal.utils.ANSI;

TerminalConnection conn = new TerminalConnection();

// Check if terminal supports synchronized output
if (conn.terminal().supportsSynchronizedOutput()) {
    conn.terminal().enableSynchronizedOutput();

    // All writes here are buffered by the terminal
    conn.write("Line 1\n");
    conn.write("Line 2\n");
    conn.write("Line 3\n");

    conn.terminal().disableSynchronizedOutput();
    // Terminal paints all three lines at once
}
```

## Terminal Support

Mode 2026 is supported by the following terminals:

| Terminal | Supported | Notes |
|----------|-----------|-------|
| Kitty | Yes | Since early versions |
| Ghostty | Yes | Since 1.0.0 |
| WezTerm | Yes | Full support |
| foot | Yes | Full support |
| Contour | Yes | Origin of the specification |
| iTerm2 | Yes | Full support |
| Mintty | Yes | Full support |
| xterm | No | Sequences ignored safely |
| GNOME Terminal | No | Sequences ignored safely |
| Konsole | No | Sequences ignored safely |
| Alacritty | No | Sequences ignored safely |

On unsupported terminals, the BSU/ESU sequences are silently ignored, so applications
can always send them without feature detection.

## Connection API

Synchronized output methods are on `TerminalFeatures`, accessed via `connection.terminal()`:

### Checking Support

```java
// Heuristic check based on terminal type detection
boolean supported = connection.terminal().supportsSynchronizedOutput();
```

This uses environment-based terminal detection via `TerminalEnvironment` and the
`Device.TerminalType` enum. No terminal query is sent.

### Runtime Query (DECRPM)

For authoritative detection, query the terminal directly using DECRQM/DECRPM:

```java
// Send CSI ? 2026 $ p and parse the DECRPM response
// Returns true (supported), false (not supported), or null (timeout)
Boolean result = connection.terminal().querySynchronizedOutput(500);

if (Boolean.TRUE.equals(result)) {
    // Terminal definitively supports Mode 2026
}
```

The DECRPM response `CSI ? 2026 ; Ps $ y` uses these Ps values:

| Ps | Meaning | Result |
|----|---------|--------|
| 0 | Not recognized | `false` |
| 1 | Set (enabled) | `true` |
| 2 | Reset (recognized but disabled) | `true` |
| 3 | Permanently set | `true` |
| 4 | Permanently reset | `false` |

### Enable / Disable

```java
connection.terminal().enableSynchronizedOutput();   // Send ESC[?2026h
connection.terminal().disableSynchronizedOutput();  // Send ESC[?2026l
```

## Automatic Readline Integration

When using the `Readline` class, synchronized output is automatically applied. If the
terminal supports Mode 2026, readline wraps prompt drawing, action execution, and resize
redraws with BSU/ESU sequences. No application code is needed.

```java
Readline readline = new Readline();

// Synchronized output is enabled automatically for supporting terminals
readline.readline(connection, new Prompt("$ "), input -> {
    // handle input
});
```

### Opting Out

To disable automatic synchronized output, set the `NO_SYNCHRONIZED_OUTPUT` flag:

```java
import org.aesh.readline.ReadlineRequest;
import org.aesh.readline.ReadlineFlag;
import org.aesh.readline.ReadlineFlags;

readline.readline(ReadlineRequest.builder()
        .connection(connection)
        .prompt("$ ")
        .requestHandler(input -> { /* handle input */ })
        .flags(ReadlineFlags.of(ReadlineFlag.NO_SYNCHRONIZED_OUTPUT))
        .build());
```

## Manual Usage

For applications that manage their own rendering loop (outside of `Readline`), use
the `Connection` methods directly or the ANSI constants:

```java
// Begin synchronized update — terminal starts buffering
connection.terminal().enableSynchronizedOutput();

// Each write is buffered by the terminal, not painted yet
connection.write("\u001B[2J");           // Clear screen
connection.write("\u001B[1;1H");         // Move to top-left
connection.write("Header line");
connection.write("\u001B[2;1H");
connection.write("Content line");
// ... more rendering ...

// End synchronized update — terminal paints everything at once
connection.terminal().disableSynchronizedOutput();
```

There is no need to batch writes into a `StringBuilder` — the terminal itself
buffers all output between BSU and ESU. Multiple `write()` calls are fine.

### ANSI Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `ANSI.MODE_2026_QUERY` | `ESC[?2026$p` | DECRQM query |
| `ANSI.MODE_2026_ENABLE` | `ESC[?2026h` | Begin synchronized update (BSU) |
| `ANSI.MODE_2026_DISABLE` | `ESC[?2026l` | End synchronized update (ESU) |

### DECRPM Response Parser

Parse a raw terminal DECRPM response:

```java
// Parse ESC[?2026;Ps$y response
String response = "\u001B[?2026;2$y";
int[] codePoints = response.codePoints().toArray();

Boolean supported = ANSI.parseMode2026Response(codePoints);
// true for Ps=1,2,3; false for Ps=0,4; null for invalid
```

The parser handles leading noise and trailing data, which is common in real terminal I/O.

## Example: Bouncing Image Demo

The `SyncImageDemoExample` demonstrates synchronized output with inline image display.
An image bounces around the screen like a screensaver — tearing is immediately visible
when synchronized output is toggled off.

```bash
mvn exec:java -pl terminal-tty \
  -Dexec.mainClass="org.aesh.terminal.tty.example.SyncImageDemoExample" \
  -Dexec.args="/path/to/image.jpg"
```

| Key | Action |
|-----|--------|
| `S` | Toggle synchronized output on/off |
| `+` / `-` | Zoom image in/out |
| `<` / `>` | Decrease/increase bounce speed |
| `Q` | Quit |

Source: [`SyncImageDemoExample.java`](https://github.com/aeshell/aesh-readline/blob/master/terminal-tty/src/main/java/org/aesh/terminal/tty/example/SyncImageDemoExample.java)

There is also a simpler dashboard demo without images:

```bash
mvn exec:java -pl terminal-tty \
  -Dexec.mainClass="org.aesh.terminal.tty.example.SynchronizedOutputExample"
```

Source: [`SynchronizedOutputExample.java`](https://github.com/aeshell/aesh-readline/blob/master/terminal-tty/src/main/java/org/aesh/terminal/tty/example/SynchronizedOutputExample.java)

## Comparison with Mode 2027

Mode 2026 (synchronized output) and Mode 2027 (grapheme cluster segmentation) are
independent features that address different problems:

| | Mode 2026 | Mode 2027 |
|-|-----------|-----------|
| **Purpose** | Prevent screen tearing | Correct cursor positioning for complex characters |
| **Scope** | Per-frame toggle (BSU/ESU) | Persistent mode (enable once at startup) |
| **Cleanup** | None needed — each ESU completes | Must disable before exit |
| **Effect** | Terminal buffers output | Terminal uses UAX #29 segmentation |

Both modes are automatically managed by `Readline` when supported.

## Specification

The synchronized output protocol was originally specified by the Contour terminal:

[Contour VT Extension: Synchronized Output](https://contour-terminal.org/vt-extensions/synchronized-output/)

## See Also

- [Connection](connection) — Full `Connection` API reference
- [Terminal Images](terminal-images) — Inline image display
- [Terminal Environment](terminal-environment) — Terminal type detection
