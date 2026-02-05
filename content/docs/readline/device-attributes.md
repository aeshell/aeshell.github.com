---
date: '2026-02-05T12:00:00+01:00'
draft: false
title: 'Device Attributes'
weight: 10
---

Æsh Readline provides support for querying terminal device attributes using DA1 and DA2 escape sequences. This allows applications to detect terminal capabilities that cannot be determined from terminfo alone.

## Overview

Device Attributes (DA) queries are escape sequences that ask the terminal to report its capabilities:

| Query | Sequence | Returns |
|-------|----------|---------|
| **DA1** (Primary) | `ESC [ c` | Device class and supported features |
| **DA2** (Secondary) | `ESC [ > c` | Terminal type and firmware version |

These queries provide authoritative information about what the terminal supports, including:
- Sixel graphics support
- ANSI color support
- Mouse/locator support
- Rectangular editing operations
- Terminal type and version

## Quick Start

```java
import org.aesh.terminal.DeviceAttributes;
import org.aesh.terminal.tty.TerminalConnection;

TerminalConnection connection = new TerminalConnection();
connection.openNonBlocking();

// Query device attributes (500ms timeout)
DeviceAttributes da = connection.queryDeviceAttributes(500);

if (da != null) {
    System.out.println("Device class: " + da.getDeviceClass());
    System.out.println("Supports Sixel: " + da.supportsSixel());
    System.out.println("Supports ANSI color: " + da.supportsAnsiColor());
    System.out.println("Supports mouse: " + da.supportsMouse());
}

connection.close();
```

## Querying Device Attributes

### Primary Device Attributes (DA1)

DA1 returns the device conformance level and supported features:

```java
DeviceAttributes da = connection.queryPrimaryDeviceAttributes(500);

if (da != null && da.hasDA1()) {
    // Device class indicates conformance level
    int deviceClass = da.getDeviceClass();
    // 1 or 2 = VT100
    // 62 = VT220
    // 63 = VT320
    // 64 = VT420

    // Check specific features
    boolean hasSixel = da.supportsSixel();
    boolean hasAnsiColor = da.supportsAnsiColor();
    boolean hasMouse = da.supportsMouse();
    boolean hasRectEdit = da.supportsRectangularEditing();
    boolean has132Cols = da.supports132Columns();

    // Get all features
    Set<DeviceAttributes.Feature> features = da.getFeatures();
}
```

### Secondary Device Attributes (DA2)

DA2 returns terminal identification and version information:

```java
DeviceAttributes da = connection.querySecondaryDeviceAttributes(500);

if (da != null && da.hasDA2()) {
    // Terminal type
    DeviceAttributes.TerminalType type = da.getTerminalType();
    System.out.println("Terminal: " + type.getName());

    // Firmware version
    int version = da.getFirmwareVersion();
    System.out.println("Version: " + version);

    // ROM cartridge (rarely used)
    int rom = da.getRomCartridge();
}
```

### Combined Query

Query both DA1 and DA2 and merge the results:

```java
DeviceAttributes da = connection.queryDeviceAttributes(500);

if (da != null) {
    // Has data from both queries
    if (da.hasDA1()) {
        System.out.println("Class: " + da.getDeviceClass());
        System.out.println("Features: " + da.getFeatures());
    }
    if (da.hasDA2()) {
        System.out.println("Type: " + da.getTerminalType().getName());
        System.out.println("Version: " + da.getFirmwareVersion());
    }
}
```

## DeviceAttributes Class

### Feature Enum

The `DeviceAttributes.Feature` enum represents capabilities reported in DA1:

| Feature | Code | Description |
|---------|------|-------------|
| `COLUMNS_132` | 1 | 132-column mode support |
| `PRINTER` | 2 | Printer port |
| `REGIS_GRAPHICS` | 3 | ReGIS graphics |
| `SIXEL` | 4 | Sixel graphics |
| `SELECTIVE_ERASE` | 6 | Selective erase |
| `DRCS` | 7 | Downloadable character sets |
| `USER_DEFINED_KEYS` | 8 | User-defined keys |
| `NATIONAL_CHARSETS` | 9 | National replacement character sets |
| `TECHNICAL_CHARSETS` | 15 | Technical character set |
| `LOCATOR` | 16 | DEC locator (mouse) |
| `STATE_INTERROGATION` | 17 | Terminal state interrogation |
| `USER_WINDOWS` | 18 | User windows |
| `HORIZONTAL_SCROLLING` | 21 | Horizontal scrolling |
| `ANSI_COLOR` | 22 | ANSI color support |
| `RECTANGULAR_EDITING` | 28 | Rectangular editing operations |
| `ANSI_TEXT_LOCATOR` | 29 | ANSI text locator (mouse) |

Check for features using:

```java
// Convenience methods
da.supportsSixel();          // Feature.SIXEL
da.supportsAnsiColor();      // Feature.ANSI_COLOR
da.supportsMouse();          // Feature.LOCATOR or ANSI_TEXT_LOCATOR
da.supportsRectangularEditing();  // Feature.RECTANGULAR_EDITING
da.supports132Columns();     // Feature.COLUMNS_132

// Generic check
da.hasFeature(DeviceAttributes.Feature.DRCS);

// Get all features
Set<Feature> features = da.getFeatures();

// Get raw parameter codes (includes unknown codes)
Set<Integer> rawParams = da.getRawParameters();
```

### TerminalType Enum

The `DeviceAttributes.TerminalType` enum represents terminal types from DA2:

| Type | Code | Description |
|------|------|-------------|
| `VT100` | 0 | VT100 terminal |
| `VT220` | 1 | VT220 terminal |
| `VT240` | 2 | VT240 terminal |
| `VT330` | 18 | VT330 terminal |
| `VT340` | 19 | VT340 terminal |
| `VT320` | 24 | VT320 terminal |
| `VT382` | 32 | VT382 terminal |
| `VT420` | 41 | VT420 terminal |
| `VT510` | 61 | VT510 terminal |
| `VT520` | 64 | VT520 terminal |
| `VT525` | 65 | VT525 terminal |
| `UNKNOWN` | -1 | Unknown terminal type |

```java
DeviceAttributes.TerminalType type = da.getTerminalType();
System.out.println("Terminal: " + type.getName());
System.out.println("Type code: " + type.getCode());
```

### Merging Device Attributes

Combine DA1 and DA2 results:

```java
DeviceAttributes da1 = connection.queryPrimaryDeviceAttributes(500);
DeviceAttributes da2 = connection.querySecondaryDeviceAttributes(500);

// Merge the results
DeviceAttributes merged = da1.merge(da2);

// Or use the combined query method
DeviceAttributes combined = connection.queryDeviceAttributes(500);
```

## Use Cases

### Sixel Graphics Detection

Use DA1 for authoritative Sixel support detection:

```java
DeviceAttributes da = connection.queryPrimaryDeviceAttributes(500);

if (da != null && da.supportsSixel()) {
    // Terminal definitively supports Sixel
    displaySixelImage(imageData);
} else {
    // Fall back to text-based display
    displayAsciiArt(imageData);
}
```

### OSC Query Support Detection

Infer OSC query support from DA1 features:

```java
DeviceAttributes da = connection.queryPrimaryDeviceAttributes(500);

if (da != null && da.likelySupportsOscQueries()) {
    // Terminal likely supports OSC 10/11 color queries
    int[] bgColor = connection.queryBackgroundColor(500);
    if (bgColor != null) {
        boolean isDark = (bgColor[0] + bgColor[1] + bgColor[2]) / 3 < 128;
    }
}

// Or use the convenience method
if (connection.supportsOscQueries(da)) {
    // Safe to use OSC queries
}
```

### Image Protocol Detection

Combine DA1 with heuristic detection for best results:

```java
// Query-based detection uses DA1 for Sixel
ImageProtocol protocol = connection.queryImageProtocol(500);

// The ImageProtocolDetector combines:
// 1. Environment variables (for Kitty/iTerm2)
// 2. TERM type string
// 3. DA1 attributes (for authoritative Sixel detection)
```

### Terminal Capabilities Report

Generate a report of terminal capabilities:

```java
DeviceAttributes da = connection.queryDeviceAttributes(500);

if (da != null) {
    StringBuilder report = new StringBuilder();
    report.append("Terminal Capabilities Report\n");
    report.append("============================\n\n");

    if (da.hasDA1()) {
        report.append("Device Class: ").append(da.getDeviceClass()).append("\n");
        report.append("Features:\n");
        for (DeviceAttributes.Feature f : da.getFeatures()) {
            report.append("  - ").append(f.name()).append("\n");
        }
    }

    if (da.hasDA2()) {
        report.append("\nTerminal Type: ").append(da.getTerminalType().getName()).append("\n");
        report.append("Firmware Version: ").append(da.getFirmwareVersion()).append("\n");
    }

    connection.write(report.toString());
}
```

## Common DA1 Response Patterns

Different terminals return different DA1 responses:

| Terminal | Response Example | Features |
|----------|------------------|----------|
| xterm | `ESC[?64;1;2;6;9;15;18;21;22c` | VT420 class, many features |
| VT220 | `ESC[?62;1;2;6;7;8;9c` | VT220 class, basic features |
| xterm+sixel | `ESC[?64;4;22c` | Sixel + ANSI color |
| Basic | `ESC[?1;0c` | VT100 class, no features |

## Error Handling

Device attribute queries may fail or timeout:

```java
DeviceAttributes da = connection.queryPrimaryDeviceAttributes(500);

if (da == null) {
    // Query failed or timed out
    // Possible reasons:
    // - Terminal doesn't support DA queries
    // - Connection not actively reading input
    // - Timeout too short

    // Fall back to heuristic detection
    ImageProtocol protocol = connection.device().getImageProtocol();
}
```

## Performance Considerations

1. **Timeout**: Choose an appropriate timeout (100-500ms typically sufficient)
2. **Cache results**: Query once at startup, not repeatedly
3. **Non-blocking**: Queries require active input reading via `openBlocking()` or `openNonBlocking()`

```java
// Good: Query once at startup
public class MyApp {
    private static DeviceAttributes terminalCapabilities;

    public static void main(String[] args) {
        TerminalConnection conn = new TerminalConnection();
        conn.openNonBlocking();

        // Query once
        terminalCapabilities = conn.queryDeviceAttributes(500);

        // Use throughout application
        if (terminalCapabilities != null && terminalCapabilities.supportsSixel()) {
            // Use Sixel graphics
        }
    }
}
```

## Supported Terminals

| Terminal | DA1 | DA2 | Notes |
|----------|-----|-----|-------|
| xterm | Yes | Yes | Full support |
| VT220+ | Yes | Yes | Original DEC terminals |
| Kitty | Yes | Limited | DA1 works, DA2 varies |
| iTerm2 | Yes | Yes | Full support |
| GNOME Terminal | Yes | Yes | Full support |
| Konsole | Yes | Yes | Full support |
| Alacritty | Yes | Limited | DA1 works |
| Windows Terminal | Yes | Yes | Full support |
| tmux | Passthrough | Passthrough | Depends on underlying terminal |

## Troubleshooting

### Query Returns Null

1. **Connection not reading**: Call `openBlocking()` or `openNonBlocking()` first
2. **Timeout too short**: Increase timeout to 500ms or more
3. **Terminal doesn't support DA**: Some terminals don't respond to DA queries

### Wrong Features Detected

1. **Terminal emulation mode**: Some terminals emulate VT100 by default
2. **SSH/tmux**: Multiplexers may filter or modify responses
3. **Check raw parameters**: Use `getRawParameters()` to see unrecognized codes

```java
// Debug: print raw response codes
Set<Integer> raw = da.getRawParameters();
System.out.println("Raw DA1 parameters: " + raw);
```
