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
| `detectFull()` | 10-50ms+ | `detect()` plus IDE settings files (JetBrains, VSCode, Windows Terminal), platform theme probing (macOS defaults, GNOME/KDE settings, Windows registry) and live terminal queries (OSC colors/palette, DA1, modes 2026/2027, sixel, grapheme clustering) — skipped under tmux/screen. |
| `detectAsync()` | returns immediately | Heuristics now, real colors in the background via a daemon thread. `awaitColors(timeout, unit)` blocks for the result; RGB accessors return `null` until it lands. Without an attached console or `/dev/tty` (pipes, containers, headless Windows), the query completes with no colors. |

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

Theme notifications handled by `EventDecoder` update the shared theme and
discard old foreground/background RGB. Other cached capabilities remain
available. Call `invalidate()` and re-detect when fresh RGB values are
needed; do standalone re-probing before starting a connection reader.

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

## Which Query Path When

Two live-query transports exist for different contexts — use, don't merge:

- **`TerminalFeatures` (in-band over the `Connection`)**: mid-session
  queries (cursor position, single OSC colors, mode probes) where a
  connection with a reader loop already exists. Responses arrive through
  the normal input pipeline (stdin-handler hijack + latch).
- **`TerminalColorQuery` via `detectFull()`/`detectAsync()`**: standalone
  probing with no connection (startup, CLIs, embedders). Opens `/dev/tty`
  directly with its own raw-mode handling — direct syscalls via FFM on
  Java 22+ (no subprocess), `stty` subprocesses on older runtimes.

### Custom probe transports

The built-in probe transport covers POSIX (`/dev/tty`, FFM syscalls on
Java 22+, `stty` otherwise) and native Windows consoles (Win32 Console
API via FFM on Java 22+). Only inject a custom transport for
environments with neither — or to override the built-ins. For ordinary
use, selection is automatic:

```java
TerminalCapabilities caps = TerminalCapabilities.detectFull();
```

`Win32ProbeTransport` is internal and cannot be constructed from an
application. To supply your own transport, implement
`TerminalProbeTransport` (raw-mode `open()` returning a
`TerminalProbeSession` with `write()`/`input()`, `close()` restoring
terminal state). The interfaces live in the zero-dependency
`terminal-detect` module, so providers elsewhere can implement them
without inverting dependencies. An injected transport replaces the
built-in one exclusively — no silent `/dev/tty` fallback that could
steal input from an embedder's reader loop. Register yours with
`TerminalCapabilities.setProbeTransport(transport)` and call
`TerminalCapabilities.invalidate()` if a result was already cached.
Pass `null` to restore the default.

### Live Windows probe check

GitHub-hosted Windows runners have no interactive console. After building
with Java 22+ from a Windows command prompt, run this **directly** in both
conhost and Windows Terminal (Surefire redirects the console handles):

```powershell
mvn -pl terminal-detect -am clean test
java --enable-native-access=ALL-UNNAMED -cp "terminal-detect\target\test-classes;terminal-detect\target\classes" org.aesh.terminal.detect.Win32ProbeLiveCheck conhost
# From Windows Terminal, use windows-terminal instead of conhost.
```

The check queries colors and grapheme clustering and verifies exact input
and output console-mode restoration. In Windows Terminal it additionally
requires an OSC background response. It fails if no real console is attached.

A live run reported on 2026-09-28 produced these results:

| Host | Shell | Console modes | Background query | Grapheme probe |
|------|-------|---------------|------------------|----------------|
| Windows Terminal | PowerShell | restored | received | true |
| Windows Terminal | cmd | restored | received | true |
| Windows Console Host | not specified | restored | no OSC background response | false |

The conhost result is a graceful no-response case, not a detected
background color. Hosted CI still cannot repeat this interactive check.

## Environment Detection Split

Emulator identity (`TerminalEnvironment`: terminal type, JetBrains/VSCode
detection, multiplexer state, OSC support) and capability inputs
(`TerminalDetector`: TERM/TERM_PROGRAM/COLORTERM/KITTY/WT_SESSION and
friends) parse overlapping variable sets by design: the former answers
*which emulator*, the latter feeds *what it supports*. They share no
code (dependency direction forbids it) — treat overlap as parallel
evolution, not a bug, unless a concrete divergence is reported.

## Related APIs

- `TerminalCapabilities` performs standalone detection (this page).
- `connection.terminal()` provides `TerminalFeatures` for live queries
  through an active connection.
- `TerminalColorCapability` in `org.aesh.terminal.utils` is the
  presentation type used by `ANSIBuilder`. Construct it with
  `TerminalColorCapability.from(TerminalCapabilities.detectFull())` at
  startup, or see [Color Detection](color-detection) for migration from
  the removed detector.
