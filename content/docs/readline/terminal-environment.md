---
date: '2026-02-06T12:00:00+01:00'
draft: false
title: 'Terminal Environment'
weight: 7
---

Æsh Readline provides centralized terminal environment detection through the `TerminalEnvironment` class. This utility parses terminal-related environment variables once and caches the results for efficient access throughout your application.

## Overview

The `TerminalEnvironment` class provides:

- **Terminal Type Detection** - Identify the terminal emulator (Kitty, iTerm2, VS Code, etc.)
- **Color Depth Detection** - Determine color capability (8, 256, or true color)
- **Multiplexer Detection** - Detect if running inside tmux or screen
- **OSC Support Detection** - Check if OSC queries are likely supported
- **Theme Hints** - Access macOS dark mode and COLORFGBG settings

All detection is done once on first access and cached for the lifetime of the JVM.

## Quick Start

```java
import org.aesh.terminal.Device;
import org.aesh.terminal.utils.TerminalEnvironment;
import org.aesh.terminal.utils.ColorDepth;

// Get the singleton instance
TerminalEnvironment env = TerminalEnvironment.getInstance();

// Detect terminal type
Device.TerminalType type = env.getTerminalType();
System.out.println("Terminal: " + type.getIdentifier());

// Check color depth
ColorDepth depth = env.getDefaultColorDepth();
System.out.println("Color depth: " + depth);

// Check for multiplexer
if (env.isInMultiplexer()) {
    System.out.println("Running inside tmux or screen");
}

// Check OSC support
if (env.supportsOscQueries()) {
    System.out.println("OSC color queries supported");
}
```

## Terminal Type Detection

The `Device.TerminalType` enum includes 24 different terminal types:

### IDE Terminals
| Type | Identifier | Color Depth |
|------|------------|-------------|
| `JETBRAINS` | JetBrains-JediTerm | True Color |
| `VSCODE` | vscode | True Color |

### macOS Terminals
| Type | Identifier | Color Depth |
|------|------------|-------------|
| `APPLE_TERMINAL` | Apple_Terminal | 256 |
| `ITERM2` | iTerm.app | True Color |

### Cross-Platform Modern Terminals
| Type | Identifier | Color Depth |
|------|------------|-------------|
| `KITTY` | kitty | True Color |
| `GHOSTTY` | ghostty | True Color |
| `ALACRITTY` | alacritty | True Color |
| `WEZTERM` | WezTerm | True Color |
| `FOOT` | foot | True Color |
| `CONTOUR` | contour | True Color |
| `RIO` | rio | True Color |
| `WARP` | warp | True Color |
| `WAVE` | wave | True Color |

### Electron-Based Terminals
| Type | Identifier | Color Depth |
|------|------------|-------------|
| `HYPER` | hyper | True Color |
| `TABBY` | tabby | True Color |
| `EXTRATERM` | extraterm | True Color |

### Linux Desktop Terminals
| Type | Identifier | Color Depth |
|------|------------|-------------|
| `GNOME_TERMINAL` | gnome-terminal | True Color |
| `KONSOLE` | konsole | True Color |
| `RXVT` | rxvt | 256 |

### Windows Terminals
| Type | Identifier | Color Depth |
|------|------------|-------------|
| `WINDOWS_TERMINAL` | Windows Terminal | True Color |
| `MINTTY` | mintty | True Color |
| `CONEMU` | ConEmu | True Color |

### Terminal Multiplexers
| Type | Identifier | Color Depth |
|------|------------|-------------|
| `TMUX` | tmux | 256 |
| `SCREEN` | screen | 256 |

### Classic/Legacy
| Type | Identifier | Color Depth |
|------|------------|-------------|
| `XTERM` | xterm | 256 |
| `LINUX_CONSOLE` | linux | 8 |
| `UNKNOWN` | unknown | 256 |

### Detection Priority

Terminal type detection follows this priority:

1. **IDE-specific environment variables** - `TERMINAL_EMULATOR` for JetBrains
2. **Terminal-specific environment variables** - `KITTY_WINDOW_ID`, `GHOSTTY_RESOURCES_DIR`, `WEZTERM_PANE`, `ITERM_SESSION_ID`, `WT_SESSION`, `ConEmuPID`, `ALACRITTY_SOCKET`
3. **TERM_PROGRAM environment variable** - Common on macOS
4. **TERM type string** - Falls back to parsing TERM value

```java
// Check for specific terminals
if (env.isKitty()) {
    System.out.println("Running in Kitty");
}

if (env.isJetBrains()) {
    System.out.println("Running in JetBrains IDE terminal");
}

if (env.isWezTerm()) {
    System.out.println("Running in WezTerm");
}
```

## OSC Code Support

Each terminal type knows which OSC (Operating System Command) codes it supports:

```java
Device.TerminalType type = env.getTerminalType();

// Check specific OSC code support
if (type.supports(Device.OscCode.FOREGROUND)) {
    // Can query foreground color (OSC 10)
}

if (type.supports(Device.OscCode.CLIPBOARD)) {
    // Can access clipboard (OSC 52)
}

// Get all supported codes
Set<Device.OscCode> codes = type.getSupportedCodes();
```

### OSC Codes

| Code | Value | Description |
|------|-------|-------------|
| `PALETTE` | 4 | Query/set palette colors |
| `FOREGROUND` | 10 | Query/set foreground color |
| `BACKGROUND` | 11 | Query/set background color |
| `CURSOR_COLOR` | 12 | Query/set cursor color |
| `CLIPBOARD` | 52 | Clipboard access |

### Terminal OSC Support Matrix

| Terminal | Palette | FG/BG | Cursor | Clipboard |
|----------|---------|-------|--------|-----------|
| Kitty | Yes | Yes | Yes | Yes |
| Ghostty | Yes | Yes | Yes | Yes |
| iTerm2 | Yes | Yes | Yes | Yes |
| WezTerm | Yes | Yes | Yes | Yes |
| Alacritty | No | Yes | Yes | No |
| VS Code | No | Yes | Yes | Yes |
| Windows Terminal | No | Yes | Yes | No |
| JetBrains | No | Yes | Yes | No |
| tmux | No | No | No | No |
| Linux Console | No | No | No | No |

## Multiplexer Detection

Detect if running inside a terminal multiplexer:

```java
// Check for any multiplexer
if (env.isInMultiplexer()) {
    System.out.println("In tmux or screen");
}

// Check specific multiplexers
if (env.isInTmux()) {
    System.out.println("In tmux");

    // Check if passthrough is enabled
    if (env.isTmuxPassthroughEnabled()) {
        System.out.println("Passthrough enabled - OSC queries may work");
    }
}

if (env.isInScreen()) {
    System.out.println("In GNU Screen");
}
```

### Nested Terminal Detection

When running inside a multiplexer, you can still detect the outer terminal:

```java
if (env.hasKnownOuterTerminal()) {
    // Running in tmux but we can detect Kitty/WezTerm/etc as outer terminal
    System.out.println("Known outer terminal detected");
}
```

This is detected via terminal-specific environment variables that persist inside multiplexers.

## Color Depth Detection

Get the terminal's color capability:

```java
ColorDepth depth = env.getDefaultColorDepth();

switch (depth) {
    case TRUE_COLOR:
        // Use 24-bit RGB colors
        System.out.println("\u001B[38;2;255;128;0mOrange\u001B[0m");
        break;
    case COLORS_256:
        // Use 256-color palette
        System.out.println("\u001B[38;5;208mOrange\u001B[0m");
        break;
    case COLORS_8:
        // Use basic ANSI colors
        System.out.println("\u001B[33mYellow\u001B[0m");
        break;
}

// Check true color support directly
if (env.isTrueColorIndicated()) {
    // COLORTERM=truecolor or COLORTERM=24bit
}
```

### Color Depth Detection Priority

1. **COLORTERM environment variable** - `truecolor` or `24bit`
2. **TERM suffix** - Contains `truecolor`, `24bit`, `256color`
3. **Terminal type** - Uses the terminal's known default
4. **OS detection** - Modern Windows (10+) supports true color

## Theme Detection

Get hints about the terminal theme:

```java
// Check macOS dark mode
if (env.isMacOsDarkMode()) {
    System.out.println("macOS Dark mode is active");
}

// Get COLORFGBG for theme hints
String colorFgBg = env.getColorFgBg();
if (colorFgBg != null) {
    // Format: "foreground;background" (e.g., "15;0" = white on black)
}
```

## Integration with Device Interface

The `Device` interface uses `TerminalEnvironment` for its default method implementations:

```java
Device device = connection.device();

// These methods use TerminalEnvironment internally
Device.TerminalType type = device.detectTerminalType();
boolean oscSupported = device.supportsOscQueries();
boolean isMultiplexer = device.isMultiplexer();
ColorDepth depth = device.getColorDepth();
```

## Static Convenience Methods

For quick access without getting the instance:

```java
// Terminal type
Device.TerminalType type = TerminalEnvironment.detectTerminalType();

// Color depth
ColorDepth depth = TerminalEnvironment.detectColorDepth();

// JetBrains check
if (TerminalEnvironment.isJetBrainsTerminal()) {
    // Handle JetBrains IDE specifics
}

// OSC support check
if (TerminalEnvironment.isOscSupported()) {
    // Safe to use OSC queries
}
```

## Refreshing the Cache

In rare cases where environment variables change during runtime:

```java
// Force re-parsing of environment variables
TerminalEnvironment.refresh();

// Get fresh values
TerminalEnvironment env = TerminalEnvironment.getInstance();
```

This is primarily useful for testing.

## Raw Environment Variable Access

Access the raw environment variable values:

```java
TerminalEnvironment env = TerminalEnvironment.getInstance();

String term = env.getTerm();              // TERM
String termProgram = env.getTermProgram(); // TERM_PROGRAM
String colorterm = env.getColorterm();     // COLORTERM
String colorFgBg = env.getColorFgBg();     // COLORFGBG
String terminalEmulator = env.getTerminalEmulator(); // TERMINAL_EMULATOR
String appleStyle = env.getAppleInterfaceStyle(); // APPLE_INTERFACE_STYLE
```

## Example: Adaptive Application

```java
import org.aesh.terminal.Device;
import org.aesh.terminal.utils.TerminalEnvironment;
import org.aesh.terminal.utils.ColorDepth;

public class AdaptiveApp {
    public static void main(String[] args) {
        TerminalEnvironment env = TerminalEnvironment.getInstance();

        System.out.println("=== Terminal Detection ===");
        System.out.println("Terminal: " + env.getTerminalType().getIdentifier());
        System.out.println("Color Depth: " + env.getDefaultColorDepth());
        System.out.println("In Multiplexer: " + env.isInMultiplexer());
        System.out.println("OSC Supported: " + env.supportsOscQueries());
        System.out.println();

        // Adapt behavior based on terminal
        Device.TerminalType type = env.getTerminalType();

        if (type == Device.TerminalType.JETBRAINS) {
            System.out.println("JetBrains IDE detected - using config file detection");
        } else if (env.isInMultiplexer() && !env.isTmuxPassthroughEnabled()) {
            System.out.println("In multiplexer without passthrough - using heuristics");
        } else if (env.supportsOscQueries()) {
            System.out.println("OSC queries supported - can detect colors");
        }

        // Adapt colors based on capability
        ColorDepth depth = env.getDefaultColorDepth();
        if (depth.supportsTrueColor()) {
            System.out.println("\u001B[38;2;0;200;100mTrue color green\u001B[0m");
        } else if (depth.supports256Colors()) {
            System.out.println("\u001B[38;5;34mPalette green\u001B[0m");
        } else {
            System.out.println("\u001B[32mBasic green\u001B[0m");
        }
    }
}
```

## Performance

- **First access**: ~1ms (parses environment variables)
- **Subsequent access**: < 0.001ms (returns cached instance)
- **Thread-safe**: All methods are safe to call from multiple threads

## Relationship to Other APIs

| Class | Purpose | Uses TerminalEnvironment? |
|-------|---------|---------------------------|
| `Device` | Terminal capabilities interface | Yes (default methods) |
| `TerminalColorCapability` | Color detection | Yes |
| `ImageProtocolDetector` | Image protocol detection | Yes |
| `TerminalColorDetector` | Full color detection with queries | Yes (for heuristics) |

`TerminalEnvironment` provides the heuristic detection layer, while `DeviceAttributes` (via DA1/DA2 queries) provides authoritative detection. Use both for best results:

```java
// Heuristic detection (fast, always available)
Device.TerminalType heuristicType = TerminalEnvironment.detectTerminalType();

// Authoritative detection (accurate, requires terminal query)
DeviceAttributes da = connection.queryDeviceAttributes(500);
Device.TerminalType authoritativeType = da.inferTerminalType();

// Validate heuristic against authoritative
if (!da.matchesTerminalType(heuristicType)) {
    System.out.println("Warning: Terminal capabilities differ from expected");
}
```
