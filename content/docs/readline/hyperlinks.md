---
date: '2026-02-26T15:00:00+01:00'
draft: false
title: 'Hyperlinks'
weight: 14
---

Æsh Readline supports clickable hyperlinks in terminal output using the OSC 8 escape sequence. When a supported terminal renders OSC 8 sequences, text is displayed as a clickable link — similar to HTML anchor tags. On terminals that don't recognize OSC 8, the sequences are silently ignored and only the visible text is shown.

## How It Works

OSC 8 uses two escape sequences to bracket the clickable text:

| Sequence | Purpose |
|----------|---------|
| `ESC ] 8 ; params ; URL ST` | Start hyperlink (associate URL with following text) |
| `ESC ] 8 ; ; ST` | End hyperlink (stop linking) |

Everything written between the start and end sequences is rendered as a clickable link. The `params` field is optional and can contain an `id=VALUE` parameter to group multiple link segments (e.g., a link that wraps across lines).

The String Terminator (ST) can be either `ESC \` or `BEL` (`\u0007`). Æsh uses `ESC \` by default.

## Quick Start

```java
import org.aesh.terminal.tty.TerminalConnection;
import org.aesh.terminal.utils.ANSI;

TerminalConnection conn = new TerminalConnection();

// Check if terminal supports hyperlinks
if (conn.terminal().supportsHyperlinks()) {
    // Write a clickable link
    conn.terminal().writeHyperlink("https://aeshell.github.io", "Æsh Documentation");
    conn.write("\n");
}
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
| Wave | Yes |
| Hyper | Yes |
| Tabby | Yes |
| Extraterm | Yes |
| GNOME Terminal | Yes |
| Konsole | Yes |
| Mintty | Yes |
| xterm | Yes |
| JetBrains | Yes |
| VS Code | Yes |
| Alacritty | Yes |
| Windows Terminal | Yes |
| Apple Terminal | No |
| rxvt | No |
| ConEmu | No |
| tmux | No |
| GNU Screen | No |
| Linux Console | No |

On unsupported terminals the OSC 8 sequences are silently ignored, so it is always safe to send them.

## Connection API

Hyperlink methods are on `TerminalFeatures`, accessed via `connection.terminal()`:

### Checking Support

```java
// Heuristic check based on terminal type detection
boolean supported = connection.terminal().supportsHyperlinks();
```

This uses environment-based terminal detection via `TerminalEnvironment` and the `Device.TerminalType` enum. No terminal query is sent.

### Writing Hyperlinks

```java
TerminalFeatures terminal = connection.terminal();

// Simple hyperlink
terminal.writeHyperlink("https://example.com", "Click here");

// Hyperlink with grouping ID (for links that span multiple lines)
terminal.writeHyperlink("https://example.com", "Click here", "link1");
```

## ANSI Utility Methods

The `ANSI` class provides static methods for building hyperlink escape sequences directly:

### Wrapping Text

```java
import org.aesh.terminal.utils.ANSI;

// Full hyperlink: opening sequence + text + closing sequence
String link = ANSI.hyperlink("https://example.com", "click here");
connection.write(link);

// With grouping ID
String link = ANSI.hyperlink("https://example.com", "click here", "myid");
```

### Building Sequences Manually

For more control, build the opening and closing sequences separately:

```java
// Opening sequence
String start = ANSI.buildHyperlinkStart("https://example.com", null);
// Result: ESC ] 8 ; ; https://example.com ST

// Opening sequence with ID
String start = ANSI.buildHyperlinkStart("https://example.com", "link1");
// Result: ESC ] 8 ; id=link1 ; https://example.com ST

// Closing sequence
String end = ANSI.buildHyperlinkEnd();
// Result: ESC ] 8 ; ; ST

// Combine manually
connection.write(start + "visible text" + end);
```

### Grouping with IDs

The `id` parameter allows multiple disjoint text segments to refer to the same hyperlink. This is useful when a link wraps across lines — hovering over any segment highlights all segments with the same ID:

```java
// Two segments of the same link on different lines
String id = "doc-link";
connection.write(ANSI.buildHyperlinkStart("https://example.com/long-page", id));
connection.write("This link continues");
connection.write(ANSI.buildHyperlinkEnd());
connection.write("\n");
connection.write(ANSI.buildHyperlinkStart("https://example.com/long-page", id));
connection.write("on the next line");
connection.write(ANSI.buildHyperlinkEnd());
```

## Security

The implementation sanitizes URLs and IDs to prevent OSC injection. Control characters that could terminate the escape sequence prematurely — specifically ESC (`\u001B`) and BEL (`\u0007`) — are stripped from both URL and ID values. Passing a `null` URL throws an `IllegalArgumentException`.

## Stripping Hyperlinks

The `Parser.stripAwayAnsiCodes()` method removes OSC 8 sequences while preserving the visible text. This is useful for measuring display width or logging plain text:

```java
import org.aesh.terminal.utils.Parser;

String linked = ANSI.hyperlink("https://example.com", "click here");
String plain = Parser.stripAwayAnsiCodes(linked);
// Result: "click here"
```

Both `ESC \` and `BEL` terminators are handled correctly.

## Complete Example

```java
import org.aesh.terminal.tty.TerminalConnection;
import org.aesh.terminal.utils.ANSI;

public class HyperlinkDemo {
    public static void main(String[] args) throws Exception {
        TerminalConnection conn = new TerminalConnection();

        if (conn.terminal().supportsHyperlinks()) {
            conn.write("Terminal supports hyperlinks!\n\n");

            // Simple link
            conn.write("Visit: ");
            conn.terminal().writeHyperlink("https://aeshell.github.io", "Æsh Documentation");
            conn.write("\n");

            // Link with styled text (combine with ANSI formatting)
            conn.write("Source: ");
            conn.write(ANSI.BOLD);
            conn.terminal().writeHyperlink("https://github.com/aeshell", "GitHub");
            conn.write(ANSI.RESET);
            conn.write("\n");

            // Multiple links on one line
            conn.write("\nLinks: ");
            conn.terminal().writeHyperlink("https://example.com/one", "One");
            conn.write(" | ");
            conn.terminal().writeHyperlink("https://example.com/two", "Two");
            conn.write(" | ");
            conn.terminal().writeHyperlink("https://example.com/three", "Three");
            conn.write("\n");
        } else {
            conn.write("This terminal does not support hyperlinks.\n");
            conn.write("Try running in: Kitty, iTerm2, WezTerm, or VS Code\n");
        }

        conn.close();
    }
}
```

## Specification

The OSC 8 hyperlink protocol is documented by the Contour terminal:

[Contour VT Extension: Clickable Links](https://contour-terminal.org/vt-extensions/clickable-links/)

## See Also

- [Connection](connection) — Full `Connection` API reference
- [Terminal Environment](terminal-environment) — Terminal type detection
- [Synchronized Output](synchronized-output) — Tear-free rendering
