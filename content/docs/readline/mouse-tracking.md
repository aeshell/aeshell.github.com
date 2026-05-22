---
date: '2026-05-20T12:00:00+02:00'
draft: false
title: 'Mouse Tracking'
weight: 12
---

Æsh Readline supports receiving mouse events from the terminal. When mouse tracking is enabled, the terminal sends escape sequences for button presses, releases, drags, hovers, and scroll wheel events. These are parsed into structured `MouseEvent` objects and dispatched to a handler on the `Connection`.

## Quick Start

```java
import org.aesh.terminal.Connection;
import org.aesh.terminal.tty.MouseEvent;
import org.aesh.terminal.tty.MouseTracking;

// Enable mouse tracking
MouseTracking.enable(connection, MouseTracking.Protocol.NORMAL);
MouseTracking.enableEncoding(connection, MouseTracking.Encoding.SGR);

// Register a handler
connection.setMouseHandler(event -> {
    System.out.printf("%s %s at (%d, %d)%n",
            event.type(), event.button(), event.x(), event.y());
});

// ... when done, disable tracking
MouseTracking.disable(connection, MouseTracking.Protocol.NORMAL);
MouseTracking.disableEncoding(connection, MouseTracking.Encoding.SGR);
```

## Mouse Event

A `MouseEvent` contains:

| Property | Type | Description |
|----------|------|-------------|
| `type()` | `Type` | PRESS, RELEASE, DRAG, MOVE, SCROLL |
| `button()` | `Button` | LEFT, MIDDLE, RIGHT, SCROLL_UP, SCROLL_DOWN, NONE |
| `x()` | `int` | Column (1-based) |
| `y()` | `int` | Row (1-based) |
| `shift()` | `boolean` | Shift modifier held |
| `alt()` | `boolean` | Alt/Meta modifier held |
| `ctrl()` | `boolean` | Ctrl modifier held |

## Protocols

The protocol controls which mouse events the terminal reports:

| Protocol | DEC Mode | Events |
|----------|----------|--------|
| `X10` | 9 | Button press only |
| `NORMAL` | 1000 | Press + release |
| `BUTTON_MOTION` | 1002 | Press + release + drag |
| `ANY_MOTION` | 1003 | Press + release + drag + hover |

`NORMAL` is the most common choice for interactive applications. `ANY_MOTION` adds hover tracking but generates high-frequency events.

## Encodings

The encoding controls how mouse coordinates are transmitted:

| Encoding | DEC Mode | Format | Limits |
|----------|----------|--------|--------|
| `SGR` | 1006 | `CSI < Pb;Px;Py M/m` | None (recommended) |
| `UTF8` | 1005 | UTF-8 encoded bytes | Deprecated |
| `URXVT` | 1015 | `CSI Pb;Px;Py M` | No release button ID |
| `SGR_PIXELS` | 1016 | Like SGR, pixel coords | Extension |

**SGR is recommended.** It has no coordinate limits (supports terminals wider/taller than 223 columns/rows), and distinguishes press (`M`) from release (`m`) with explicit button identification.

## How It Works

### Enabling

`MouseTracking.enable()` writes a DEC private mode sequence to the terminal:

```
CSI ? 1000 h     (enable normal tracking)
CSI ? 1006 h     (enable SGR encoding)
```

### Event Flow

When the user clicks at column 42, row 17:

1. Terminal sends: `ESC [ < 0 ; 42 ; 17 M`
2. `EventDecoder` intercepts the sequence (when a mouse handler is registered)
3. The SGR parameters are parsed: button byte = 0, x = 42, y = 17
4. A `MouseEvent` is created and dispatched to the handler
5. The sequence is consumed -- it does not reach `ActionDecoder` or the readline engine

### Button Byte Decoding

The button byte (`Pb`) is a bitfield:

| Bits | Mask | Meaning |
|------|------|---------|
| 0-1 | `0x03` | Button index: 0=left, 1=middle, 2=right, 3=none |
| 2 | `0x04` | Shift modifier |
| 3 | `0x08` | Alt/Meta modifier |
| 4 | `0x10` | Ctrl modifier |
| 5 | `0x20` | Motion flag (drag or move) |
| 6 | `0x40` | Scroll wheel (0=up, 1=down in bits 0-1) |

### Disabling

Mouse tracking should be disabled when the application exits or when mouse input is no longer needed. This restores normal terminal behavior:

```
CSI ? 1000 l     (disable normal tracking)
CSI ? 1006 l     (disable SGR encoding)
```

## Connection Integration

The mouse handler is set on the `Connection`, following the same pattern as signal and theme handlers:

```java
// Set handler
connection.setMouseHandler(event -> { ... });

// Get current handler
Consumer<MouseEvent> handler = connection.mouseHandler();

// Remove handler (mouse sequences pass through as input)
connection.setMouseHandler(null);
```

When no mouse handler is registered, mouse escape sequences pass through to the input handler unchanged. The `ActionDecoder` will consume them as `SequenceKeyAction` via the [VtParser](vt-parser) fallback, preventing byte-by-byte leakage.

## Windows Support

On Windows, mouse events are delivered through the VT input path when `ENABLE_VIRTUAL_TERMINAL_INPUT` is active. The `MouseTracking.enable()` / `enableEncoding()` sequences tell the console to start sending SGR mouse reports, which are intercepted by the `EventDecoder` mouse filter.

When `connection.setMouseHandler()` is called on a `TerminalConnection` backed by a Windows terminal, it automatically:
- Enables `ENABLE_MOUSE_INPUT` on the console input handle
- Disables `ENABLE_QUICK_EDIT_MODE` (which conflicts with mouse tracking)

### Known Limitations on Windows

- **SHIFT modifier is not detected.** The Windows console does not encode the SHIFT bit in the SGR mouse button byte. This affects all SGR-based mouse tracking on Windows, not just Æsh Readline. Ctrl and Alt modifiers work correctly.
- **Legacy consoles** without VT input support may not receive mouse events via the SGR path.

## Terminal Support

Most modern terminal emulators support SGR mouse encoding:

| Terminal | SGR Mouse | Notes |
|----------|-----------|-------|
| xterm | Yes | Origin of the protocol |
| Ghostty | Yes | |
| Kitty | Yes | |
| Alacritty | Yes | |
| WezTerm | Yes | |
| iTerm2 | Yes | |
| GNOME Terminal (VTE) | Yes | |
| Windows Terminal | Yes | Via VT input mode |
| Windows Console | Yes | Requires VT input support (Windows 10 1809+) |
| tmux | Yes | Requires `set -g mouse on` |
| Linux console | No | No mouse support |

Use `Device.TerminalType.expectsMouse()` or `DeviceAttributes.supportsMouse()` to check mouse support programmatically.

## Example: Mouse-Driven Selection

```java
MouseTracking.enable(connection, MouseTracking.Protocol.BUTTON_MOTION);
MouseTracking.enableEncoding(connection, MouseTracking.Encoding.SGR);

int startX = -1, startY = -1;

connection.setMouseHandler(event -> {
    switch (event.type()) {
        case PRESS:
            startX = event.x();
            startY = event.y();
            break;
        case DRAG:
            // Highlight from (startX, startY) to (event.x(), event.y())
            highlightRegion(startX, startY, event.x(), event.y());
            break;
        case RELEASE:
            // Selection complete
            String selected = getTextInRegion(startX, startY, event.x(), event.y());
            break;
    }
});
```

## See Also

- [VT Escape Sequence Parser](vt-parser) -- classifies mouse CSI sequences
- [Connection](connection) -- handler registration and lifecycle
- [Device Attributes](device-attributes) -- `supportsMouse()` capability detection
- [Terminal Environment](terminal-environment) -- `expectsMouse()` heuristic detection
