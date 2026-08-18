---
date: '2026-05-20T12:00:00+02:00'
draft: false
title: 'VT Escape Sequence Parser'
weight: 11
---

The `VtParser` is a general-purpose ANSI/VT escape sequence parser based on [Paul Flo Williams' DEC-compatible parser](https://vt100.net/emu/dec_ansi_parser). It classifies all terminal input into structured events: printable characters, control characters, CSI sequences, OSC strings, DCS passthrough, and escape sequences.

## Overview

Terminal input contains a mix of printable text and escape sequences (key codes, mouse events, color query responses, device attribute reports). The `VtParser` classifies each byte using a pre-computed lookup table, guaranteeing that every byte in every state has a defined transition.

The parser is used internally by `ActionDecoder` to prevent unknown escape sequences from leaking byte-by-byte into the input stream. It can also be used directly by applications that need to process raw terminal responses.

## Quick Start

```java
import org.aesh.terminal.parser.VtParser;
import org.aesh.terminal.parser.VtHandler;

VtParser parser = new VtParser(new VtHandler() {
    @Override
    public void print(int codePoint) {
        System.out.print((char) codePoint);
    }

    @Override
    public void csiDispatch(int finalChar, int[] params, int paramCount,
                            int[] intermediates, int intermediateCount,
                            boolean hasSubParams) {
        System.out.printf("CSI sequence: final=%c params=%d%n",
                (char) finalChar, paramCount);
    }

    @Override
    public void oscEnd(String data) {
        System.out.println("OSC: " + data);
    }
});

// Process input
byte[] input = "\033[2JHello\033]0;title\007".getBytes();
parser.advance(input, 0, input.length);
```

## How It Works

The parser uses a `short[14][256]` transition table (~7KB) where each entry encodes `(action << 4 | nextState)`. This is a single array lookup per byte with no branching -- the table is the state machine.

### States

The 14 states follow the Williams specification:

| State | Purpose |
|-------|---------|
| `ground` | Initial state. Printable characters are dispatched via `print()`. |
| `escape` | After ESC (0x1B). Routes to CSI, OSC, DCS, or simple escape sequence. |
| `escape_intermediate` | Collecting intermediate characters (0x20-0x2F) in an escape sequence. |
| `csi_entry` | After CSI. Handles first character (private marker or parameter). |
| `csi_param` | Collecting CSI parameters (digits, semicolons, colons). |
| `csi_intermediate` | Collecting CSI intermediate characters. |
| `csi_ignore` | Consuming a malformed CSI sequence (discards without dispatching). |
| `dcs_entry` | After DCS. Mirrors CSI entry structure. |
| `dcs_param` | Collecting DCS parameters. |
| `dcs_intermediate` | Collecting DCS intermediate characters. |
| `dcs_passthrough` | Passing DCS data to handler via `put()`. |
| `dcs_ignore` | Consuming a malformed DCS string. |
| `osc_string` | Collecting OSC string content. |
| `sos_pm_apc_string` | Ignoring SOS/PM/APC strings (no DEC-defined functions). |

### Actions

| Action | Callback | When |
|--------|----------|------|
| `print` | `VtHandler.print(codePoint)` | Printable character in ground state |
| `execute` | `VtHandler.execute(controlChar)` | C0/C1 control character |
| `csi_dispatch` | `VtHandler.csiDispatch(...)` | Complete CSI sequence |
| `esc_dispatch` | `VtHandler.escDispatch(...)` | Complete escape sequence |
| `osc_end` | `VtHandler.oscEnd(data)` | Complete OSC string |
| `hook` | `VtHandler.hook(...)` | DCS passthrough started |
| `put` | `VtHandler.put(byte)` | DCS data byte |
| `unhook` | `VtHandler.unhook()` | DCS passthrough ended |

### Deviations from Williams

Two modern extensions are included:

1. **Colon (0x3A) as subparameter separator** -- Williams specifies that colon causes CSI sequences to be ignored. Modern terminals use colon for SGR extended color syntax (`38:2:R:G:B`), so the parser treats it as a valid parameter separator and sets the `hasSubParams` flag.

2. **BEL (0x07) terminates OSC strings** -- Williams specifies only ST (0x9C or ESC \\) as the OSC terminator. xterm and all modern terminals also accept BEL, which is the de facto standard.

## Input Formats

The parser accepts both raw bytes and Unicode code points:

```java
// Byte array (from terminal I/O)
parser.advance(bytes, offset, length);

// Code point array (from Decoder output in the readline pipeline)
parser.advance(codePoints, offset, length);

// Single value
parser.advance(byteOrCodePoint);
```

Code points above 255 are treated as printable in the ground state and ignored in other states.

## Integration with the Input Pipeline

VtParser is used in two places in the input pipeline:

### EventDecoder Sequence Filtering

The `EventDecoder` uses a single `VtParser` instance to classify and filter terminal sequences from the input stream. When a handler is registered (theme change, mouse, focus), the VtParser identifies matching CSI sequences via its `csiDispatch` callback:

- **Theme DSR** (`CSI ? 997 ; Ps n`) -- dispatched to the theme change handler
- **Mouse SGR** (`CSI < Pb ; Px ; Py M/m`) -- dispatched to the mouse handler
- **Focus events** (`CSI I` / `CSI O`) -- dispatched to the focus handler

Non-matching sequences are re-emitted as regular input. The VtParser naturally handles sequences split across input buffer boundaries, and a fast-path ESC scan skips VtParser processing entirely when no ESC byte is present in the input (the >99% common case for keystrokes).

For text between escape sequences, the filter uses bulk `System.arraycopy` to skip per-byte VtParser processing, keeping overhead minimal for mixed input.

### ActionDecoder Sequence Measurement

The `ActionDecoder` uses `VtParser` as a fallback when the `KeyMappingTrie` has no match for an escape sequence:

1. `KeyMappingTrie` is checked first (handles known keys like arrow keys, function keys)
2. If no match and no prefix, and the buffer starts with ESC, `VtParser` measures the complete sequence length
3. The entire sequence is consumed as a single `SequenceKeyAction`
4. If the sequence is incomplete, the parser waits for more input

This means unknown CSI sequences (mouse events, paste brackets, terminal responses) are cleanly consumed rather than corrupting the input stream.

## Performance

The table-driven design provides consistent per-byte cost regardless of sequence type:

| Input | Time |
|-------|------|
| Single printable character | ~5 ns |
| CSI arrow key (3 bytes) | ~22 ns |
| CSI SGR color (18 bytes) | ~67 ns |
| OSC title (22 bytes) | ~88 ns |
| Mixed realistic input (93 bytes) | ~231 ns (~2.5 ns/byte) |

## See Also

- [Focus Tracking](focus-tracking) -- focus events (`CSI I` / `CSI O`) are filtered by VtParser
- [Mouse Tracking](mouse-tracking) -- SGR mouse events are filtered by VtParser
- [Device Attributes](device-attributes) -- DA1/DA2 responses are CSI sequences parsed by VtParser
- Paul Flo Williams, [A parser for DEC's ANSI-compatible video terminals](https://vt100.net/emu/dec_ansi_parser)
