---
date: '2026-09-25T12:00:00+02:00'
draft: false
title: 'Terminal Capabilities'
weight: 11
---

The `terminal-detect` module answers "what can this terminal do?" with zero
runtime dependencies. Its facade is `TerminalCapabilities`
(`org.aesh.terminal.detect`); `TerminalTheme` and `ImageProtocol` describe
the answers.

## Quick Start: Three Detection Modes

| Method | Cost | What it does |
|--------|------|--------------|
| `detect()` | ~1-2ms | Environment variables only. No subprocesses, no terminal I/O. |
| `detectFull()` | 10-50ms+ | `detect()` plus platform theme probing (macOS defaults, GNOME/KDE settings, Windows registry) and live terminal queries (OSC colors/palette, DA1, modes 2026/2027, sixel, grapheme clustering) — skipped under tmux/screen. |
| `detectAsync()` | returns immediately | Heuristics now, real colors in the background via a daemon thread. `awaitColors(timeout, unit)` blocks for the result; RGB accessors return `null` until it lands. On Windows, pipes, containers, or tmux without passthrough the background query completes immediately with no results. |

```java
import org.aesh.terminal.detect.TerminalCapabilities;

// Fast path: startup, prompts, anything latency-sensitive
TerminalCapabilities caps = TerminalCapabilities.detect();
if (caps.supportsTrueColor()) { ... }

// Full path: theme-aware rendering, image protocol choice
TerminalCapabilities full = TerminalCapabilities.detectFull();
if (full.theme().isDark()) { ... }           // UNKNOWN counts as dark
int[] bg = full.backgroundRGB();             // null if unqueryable

// Background path: don't block the UI thread
TerminalCapabilities async = TerminalCapabilities.detectAsync();
if (async.awaitColors(500, TimeUnit.MILLISECONDS)) {
    int[] palette = async.paletteColor(4);
}
```

A shared instance exists for tests and embedders: `getInstance()` /
`setInstance(caps)` lets test harnesses inject fixed capabilities instead
of depending on the CI environment.

## What You Get

- **Color depth**: `supportsColor()`, `supports256Colors()`, `supportsTrueColor()`.
- **Theme**: `theme()` never returns `null` (falls back to the
  environment-derived theme, which may still be `UNKNOWN`). Note
  `isDark()` treats `UNKNOWN` as dark — safe default for choosing
  light-on-dark text, worth knowing when branching on it.
- **Colors**: `foregroundRGB()`, `backgroundRGB()`, `paletteColors()` /
  `paletteColor(index)`, plus named `black()` … `white()` and brights.
- **Images**: `imageProtocol()` — Kitty, iTerm2, Sixel, or none. See
  [Terminal Images](terminal-images) for rendering once chosen.

## Old vs New API

Two generations coexist, sharing the `TerminalTheme` enum:

- **New** (`org.aesh.terminal.detect`, this page): the detection engine.
  Zero dependencies — usable standalone, even outside aesh. Prefer for
  all new code.
- **Old** (`TerminalColorDetector` / `TerminalColorCapability` in
  `org.aesh.terminal.utils`, see [Color Detection](color-detection)):
  the previous presentation layer, still used internally (e.g. by
  `ANSIBuilder`). No migration needed for existing users; new code
  should start here.
