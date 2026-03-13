---
date: '2026-03-13T15:00:00+01:00'
draft: false
title: 'Clipboard'
weight: 15
---

Æsh Readline can automatically copy killed and yanked text to the system clipboard using the OSC 52 escape sequence. When enabled, any text that enters the internal kill ring (via Ctrl+K, Ctrl+W, vi `dd`, `yy`, etc.) is also written to the system clipboard, allowing you to paste it into other applications.

This is a **write-only** operation — Æsh never reads from the clipboard, so there are no security concerns about exposing clipboard contents.

## How It Works

OSC 52 sends Base64-encoded text to the terminal, which places it on the system clipboard:

| Sequence | Purpose |
|----------|---------|
| `ESC ] 52 ; c ; <base64> ST` | Copy text to the system clipboard |

The `c` parameter selects the clipboard buffer (as opposed to the primary selection). The String Terminator (ST) can be either `ESC \` or `BEL` (`\u0007`). Æsh uses `ESC \`.

On terminals that don't recognize OSC 52, the sequence is silently ignored — no visible output or errors.

## Quick Start

Clipboard writing is **automatic** when using the `Readline` class. If the terminal supports OSC 52, all kill/copy actions write to the system clipboard with no application code needed:

```java
Readline readline = new Readline();

// Clipboard integration is enabled automatically for supporting terminals.
// When the user presses Ctrl+K, Ctrl+W, etc., text goes to both
// the internal kill ring AND the system clipboard.
readline.readline(connection, new Prompt("$ "), input -> {
    // handle input
});
```

## Terminal Support

| Terminal | Supported |
|----------|-----------|
| iTerm2 | Yes |
| Kitty | Yes |
| Ghostty | Yes |
| WezTerm | Yes |
| foot | Yes |
| Contour | Yes |
| Rio | Yes |
| Warp | Yes |
| Mintty | Yes |
| xterm | Yes (if `allowWindowOps` is enabled) |
| GNOME Terminal | Yes |
| Konsole | Yes |
| VS Code | Yes |
| Tabby | Yes |
| Hyper | Yes |
| Windows Terminal | No |
| Alacritty | No |
| Apple Terminal | No |
| ConEmu | No |
| Linux Console | No |

On unsupported terminals the OSC 52 sequence is never sent — the `supportsClipboard()` check prevents it.

## Connection API

The `Connection` interface provides methods for clipboard access:

### Checking Support

```java
// Heuristic check based on terminal type detection
boolean supported = connection.supportsClipboard();
```

This uses environment-based terminal detection via `TerminalEnvironment` and the `Device.TerminalType` enum. No terminal query is sent.

### Writing to the Clipboard

```java
// Copy text to the system clipboard
connection.writeClipboard("Hello, clipboard!");
```

The method is a no-op if the text is null/empty or the terminal doesn't support OSC 52. It returns the `Connection` for chaining:

```java
connection
    .writeClipboard("copied text")
    .write("Text copied to clipboard!\n");
```

## ANSI Utility Methods

The `ANSI` class provides a static method for building the OSC 52 escape sequence directly:

```java
import org.aesh.terminal.utils.ANSI;

// Build an OSC 52 sequence
String seq = ANSI.buildOsc52Write("text to copy");
// Result: ESC ] 52 ; c ; dGV4dCB0byBjb3B5 ESC \

// Write it manually
connection.write(seq);
```

### Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `ANSI.OSC_CLIPBOARD` | `52` | OSC code for clipboard access |

## Automatic Readline Integration

When using the `Readline` class, clipboard writing is handled automatically through the `PasteManager`. Every call to `PasteManager.addText()` — which all kill/copy actions use — triggers a clipboard write if OSC 52 is supported.

Actions that write to the clipboard:

| Action | Key (Emacs) | Key (Vi) |
|--------|------------|----------|
| Kill to end of line | `Ctrl+K` | `D` (command mode) |
| Kill to start of line | `Ctrl+U` | — |
| Kill word backward | `Ctrl+W` | — |
| Kill word (Unix filename rubout) | — | — |
| Delete character | `Ctrl+D` | `x` |
| Delete previous character | `Backspace` | `X` |
| Copy line | — | `yy` |
| Change action | — | `c` + motion |

### Opting Out

To disable automatic clipboard writing, set the `NO_CLIPBOARD` flag:

```java
import org.aesh.readline.ReadlineFlag;

EnumMap<ReadlineFlag, Integer> flags = new EnumMap<>(ReadlineFlag.class);
flags.put(ReadlineFlag.NO_CLIPBOARD, 0);

readline.readline(connection, new Prompt("$ "), input -> {
    // handle input
}, null, null, null, null, flags);
```

## Manual Usage

For applications that manage their own rendering (outside of `Readline`), use the `Connection` method directly:

```java
// Copy selected text to the system clipboard
if (connection.supportsClipboard()) {
    connection.writeClipboard(selectedText);
}
```

Or build the sequence manually with the `ANSI` utility:

```java
String osc52 = ANSI.buildOsc52Write(selectedText);
connection.write(osc52);
```

## Specification

The OSC 52 protocol is documented in the xterm control sequences specification:

[XTerm Control Sequences — Operating System Commands](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html#h3-Operating-System-Commands)

## See Also

- [Connection](connection) — Full `Connection` API reference
- [Readline API](readline-api) — ReadlineFlag and core API reference
- [Terminal Environment](terminal-environment) — Terminal type detection
- [Hyperlinks](hyperlinks) — OSC 8 clickable links
