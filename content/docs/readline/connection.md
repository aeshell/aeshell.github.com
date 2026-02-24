---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Connection'
weight: 9
---

The `Connection` interface represents a connection to a terminal (local, direct, or remote).

## Creating Connections

### Local Terminal Connection

```java
import org.aesh.tty.terminal.TerminalConnection;

Connection connection = new TerminalConnection();
```

### Opening Connections

#### Blocking Mode

```java
connection.openBlocking();
```

Blocks the current thread until the connection is closed.

#### Non-Blocking Mode

```java
connection.openNonBlocking();
```

Reads input in a separate thread, allowing the current thread to continue.

#### Checking Reading State

```java
// Check if connection is actively reading
if (connection.reading()) {
    // Handler-based queries work (setStdinHandler)
} else {
    // Use synchronous methods like queryColorCapability()
}
```

The `reading()` method returns `true` after `openBlocking()` or `openNonBlocking()` is called
and before `close()` is called. This is useful for determining which query methods to use:
- When `reading()` is true: Handler-based methods like `queryTerminal()` work
- When `reading()` is false: Use synchronous methods like `queryColorCapability()`

## Handlers

### Standard Input Handler

```java
connection.setStdinHandler(input -> {
    for (int codePoint : input) {
        System.out.println("Char: " + (char) codePoint);
    }
});

Consumer<int[]> handler = connection.getStdinHandler();
```

### Standard Output Handler

```java
Consumer<int[]> outputHandler = connection.stdoutHandler();
outputHandler.accept("Output text\n".codePoints().toArray());

// Convenience method
connection.write("Hello, World!\n");
```

### Size Handler

Called when terminal is resized:

```java
connection.setSizeHandler(size -> {
    int width = size.getWidth();
    int height = size.getHeight();
    System.out.println("Terminal resized: " + width + "x" + height);
});

Consumer<Size> sizeHandler = connection.getSizeHandler();
```

### Signal Handler

Called when terminal signals are received:

```java
connection.setSignalHandler(signal -> {
    System.out.println("Signal: " + signal);
});

Consumer<Signal> signalHandler = connection.getSignalHandler();
```

### Close Handler

Called when the connection is closed:

```java
connection.setCloseHandler(ignored -> {
    System.out.println("Connection closed");
});

Consumer<Void> closeHandler = connection.getCloseHandler();
```

### Theme Change Handler

Called when the terminal's theme changes (dark/light mode switch). Requires subscribing to theme change notifications with `enableThemeChangeNotification()`:

```java
import org.aesh.terminal.utils.TerminalTheme;

// Set handler for theme changes
connection.setThemeChangeHandler(theme -> {
    System.out.println("Theme changed to: " + theme);
    // Re-detect colors or update cached capability
});

Consumer<TerminalTheme> themeHandler = connection.getThemeChangeHandler();
```

When a handler is registered, the `EventDecoder` in the input pipeline intercepts unsolicited `CSI ? 997 ; Ps n` responses and routes them to this handler, preventing them from appearing as input. See [Theme Mode Queries](#theme-mode-queries) for the full API.

## Terminal Properties

```java
Device device = connection.device();
Size size = connection.size();

// Encoding
Charset inputEncoding = connection.inputEncoding();
Charset outputEncoding = connection.outputEncoding();

// ANSI support
boolean supportsAnsi = connection.supportsAnsi();
```

## Attributes

```java
Attributes attributes = connection.getAttributes();
connection.setAttributes(new Attributes);
```

## Capabilities

Set terminal capabilities:

```java
boolean success = connection.put(Capability.key_x, 1);
```

## Closing Connections

```java
connection.close();          // Close with default exit
connection.close(0);        // Close with specific exit code
```

## Write Convenience

```java
Connection connection = ...;

connection.write("Hello");
connection.write("Line 1\nLine 2\n");
```

## Raw Mode

Enter raw mode for character-by-character input:

```java
Attributes previous = connection.enterRawMode();
// ... work in raw mode ...
connection.setAttributes(previous);
```

## Cursor Position

Get cursor position:

```java
Point position = connection.getCursorPosition();
int row = position.getRow();
int col = position.getColumn();
```

## OSC Queries

OSC (Operating System Command) queries allow you to interrogate the terminal for information like colors, clipboard content, and more.

### Generic OSC Query

Send any OSC query with a custom response parser:

```java
// Query palette color 4 with custom parser
String result = connection.queryOsc(4, "?", 500, input -> {
    // Custom parsing logic
    return parseColorResponse(input);
});
```

### Color Queries

Query the terminal's current colors:

```java
// Query foreground color (OSC 10)
int[] fg = connection.queryForegroundColor(500);
if (fg != null) {
    System.out.println("Foreground: RGB(" + fg[0] + "," + fg[1] + "," + fg[2] + ")");
}

// Query background color (OSC 11)
int[] bg = connection.queryBackgroundColor(500);
if (bg != null) {
    System.out.println("Background: RGB(" + bg[0] + "," + bg[1] + "," + bg[2] + ")");
}

// Query cursor color (OSC 12)
int[] cursor = connection.queryCursorColor(500);
```

### Palette Color Queries

Query colors from the 256-color palette using OSC 4:

```java
// Query palette color by index (0-255)
int[] color = connection.queryPaletteColor(1, 500);
if (color != null) {
    System.out.println("Palette 1: RGB(" + color[0] + "," + color[1] + "," + color[2] + ")");

    // Convert to nearest 256-color index
    int index = ANSI.rgbTo256Color(color[0], color[1], color[2]);

    // Convert to basic ANSI code
    int ansiCode = ANSI.rgbToAnsiColor(color[0], color[1], color[2]);
}
```

Palette indices:
- 0-7: Standard ANSI colors
- 8-15: Bright ANSI colors
- 16-231: 6x6x6 color cube
- 232-255: Grayscale ramp

### OSC Query with Index Parameter

For OSC codes that require an index (like OSC 4), use the indexed query method:

```java
// Query OSC 4 with index parameter
int[] rgb = connection.queryOsc(4, 1, "?", 500,
        input -> ANSI.parseOscColorResponse(input, 4, 1));
```

### OSC Support Detection

Not all terminals support all OSC queries. For example, JetBrains IDE terminals
don't support OSC 4 (palette queries). Use the `Device` enums to check
terminal capabilities before querying:

```java
import org.aesh.terminal.Device.TerminalType;
import org.aesh.terminal.Device.OscCode;

// Detect terminal type from environment
TerminalType termType = connection.getTerminalType();
System.out.println("Terminal: " + termType.getIdentifier());

// Check specific OSC support
if (connection.supportsPaletteQuery()) {
    int[] color = connection.queryPaletteColor(1, 500);
    // Use color...
}

// Or use the convenience method that checks support first
int[] color = connection.queryPaletteColorIfSupported(1, 500);
if (color != null) {
    // Terminal supports OSC 4 and returned a color
}

// Check for OSC 10/11 support
if (connection.supportsColorQuery()) {
    int[] fg = connection.queryForegroundColor(500);
    int[] bg = connection.queryBackgroundColor(500);
}

// Check if running in JetBrains IDE
Device device = connection.device();
if (device != null && device.isJetBrainsTerminal()) {
    // Use fallback approach for palette colors
}

// Check for any OSC code support
if (connection.supportsOscCode(OscCode.CLIPBOARD)) {
    // Terminal supports clipboard access via OSC 52
}
```

Known terminal limitations:
- **JetBrains IDEs**: No OSC 4 (palette) support
- **Linux console**: No OSC query support
- **Alacritty**: No OSC 52 (clipboard) support

### Batch OSC Queries

For better performance when querying multiple colors, use batch queries. This sends all queries at once and collects responses together, reducing latency from O(n × timeout) to O(timeout):

```java
// Query foreground, background, and cursor colors in one operation
// Takes ~50-100ms instead of ~300-400ms for individual queries
Map<Integer, int[]> colors = connection.queryColors(500);

int[] fg = colors.get(ANSI.OSC_FOREGROUND);   // OSC 10
int[] bg = colors.get(ANSI.OSC_BACKGROUND);   // OSC 11
int[] cursor = colors.get(ANSI.OSC_CURSOR_COLOR);  // OSC 12

// Query multiple palette colors at once
Map<Integer, int[]> palette = connection.queryPaletteColors(500, 0, 1, 2, 3, 4, 5, 6, 7);

// Query all 16 ANSI colors
Map<Integer, int[]> ansi16 = connection.queryAnsi16Colors(500);

// Generic batch query for any OSC codes
Map<Integer, int[]> results = connection.queryBatchOsc(500, 10, 11, 12);
```

For higher-level access with automatic fallbacks, use `TerminalColorDetector`:

```java
import org.aesh.terminal.tty.TerminalColorDetector;

// Query with automatic fallback to environment-based detection
Map<Integer, int[]> colors = TerminalColorDetector.queryColorsWithFallback(connection, 500);
// Always returns colors - actual if OSC works, estimated if not
```

## Theme Mode Queries

The `Connection` interface provides methods for querying and subscribing to terminal theme changes using the `CSI ? 996 n` protocol. This is simpler and faster than OSC 10/11 RGB queries — it returns a direct `DARK` or `LIGHT` answer.

See [Color Detection](color-detection#1-theme-mode-query-csi--996-n--fastest--simplest) for background on the protocol.

### One-Shot Query

Query the current theme mode:

```java
// Returns DARK, LIGHT, or null (unsupported/timeout)
TerminalTheme theme = connection.queryThemeMode(500);

if (theme != null) {
    System.out.println("Terminal theme: " + theme);
}
```

### Checking Support

```java
// Check if the terminal supports theme queries (avoids timeout)
if (connection.supportsThemeQuery()) {
    TerminalTheme theme = connection.queryThemeMode(500);
}
```

Support is determined by the `Device.TerminalType` enum — terminals like Kitty, Ghostty, Foot, Contour, GNOME Terminal, and tmux report `supportsThemeDsr() == true`.

### Subscribing to Live Theme Changes

Enable real-time notifications when the user switches dark/light mode:

```java
// One-call setup: register handler and enable notifications
connection.enableThemeChangeNotification(theme -> {
    System.out.println("Theme changed to: " + theme);
    // Update cached colors
    capability = new TerminalColorCapability(capability.getColorDepth(), theme);
});
```

Or step by step:

```java
// 1. Register the handler
connection.setThemeChangeHandler(theme -> {
    System.out.println("Theme changed to: " + theme);
});

// 2. Tell the terminal to send notifications (CSI ? 2031 h)
connection.enableThemeChangeNotification();
```

### Unsubscribing

```java
// Tell the terminal to stop sending notifications (CSI ? 2031 l)
connection.disableThemeChangeNotification();

// Optionally remove the handler
connection.setThemeChangeHandler(null);
```

### Supported Terminals

| Terminal | Version | Theme DSR |
|----------|---------|-----------|
| Contour | 0.4.0+ | Yes |
| Ghostty | 1.0.0+ | Yes |
| Kitty | 0.38.1+ | Yes |
| Foot | — | Yes |
| VTE / GNOME Terminal | 0.82.0+ | Yes |
| tmux | — | Yes (passthrough) |

### Complete Example

```java
import org.aesh.readline.tty.terminal.TerminalConnection;
import org.aesh.terminal.utils.TerminalTheme;

TerminalConnection connection = new TerminalConnection();

// Query current theme
if (connection.supportsThemeQuery()) {
    TerminalTheme theme = connection.queryThemeMode(500);
    System.out.println("Current theme: " + theme);

    // Subscribe to changes
    connection.enableThemeChangeNotification(newTheme -> {
        System.out.println("Theme switched to: " + newTheme);
    });
}

// ... application runs ...

// Clean up before exit
connection.disableThemeChangeNotification();
connection.setThemeChangeHandler(null);
connection.close();
```

## Synchronized Output (Mode 2026)

Synchronized output prevents screen tearing by telling the terminal to buffer all output until the frame is complete. See [Synchronized Output](synchronized-output) for full documentation.

```java
// Check support (heuristic, no query sent)
if (connection.supportsSynchronizedOutput()) {
    connection.enableSynchronizedOutput();
    // ... render frame ...
    connection.disableSynchronizedOutput();
}

// Runtime query via DECRPM (authoritative)
Boolean supported = connection.querySynchronizedOutput(500);
// true = supported, false = not supported, null = timeout
```

Synchronized output is automatically managed by `Readline` for supporting terminals. Use the `ReadlineFlag.NO_SYNCHRONIZED_OUTPUT` flag to opt out.

## Device Attributes (DA1/DA2)

Device Attributes queries allow you to detect terminal capabilities that cannot be determined from terminfo alone.

### Primary Device Attributes (DA1)

Query the terminal's conformance level and supported features:

```java
DeviceAttributes da = connection.queryPrimaryDeviceAttributes(500);

if (da != null) {
    // Device class (1=VT100, 62=VT220, 64=VT420, etc.)
    int deviceClass = da.getDeviceClass();

    // Check specific features
    boolean hasSixel = da.supportsSixel();
    boolean hasAnsiColor = da.supportsAnsiColor();
    boolean hasMouse = da.supportsMouse();
    boolean has132Cols = da.supports132Columns();

    // Check any feature by enum
    if (da.hasFeature(DeviceAttributes.Feature.RECTANGULAR_EDITING)) {
        // Terminal supports rectangular editing operations
    }
}
```

### Secondary Device Attributes (DA2)

Query terminal identification and version information:

```java
DeviceAttributes da = connection.querySecondaryDeviceAttributes(500);

if (da != null) {
    // Terminal type (VT100, VT220, VT420, etc.)
    DeviceAttributes.TerminalType type = da.getTerminalType();

    // Firmware/version number
    int version = da.getFirmwareVersion();

    System.out.println("Terminal: " + type.getName() + " v" + version);
}
```

### Combined Query

Query both DA1 and DA2 and merge the results:

```java
DeviceAttributes da = connection.queryDeviceAttributes(500);

if (da != null) {
    // Has both DA1 and DA2 data
    System.out.println("Class: " + da.getDeviceClass());
    System.out.println("Type: " + da.getTerminalType().getName());
    System.out.println("Features: " + da.getFeatures());
}
```

### Available Features

The `DeviceAttributes.Feature` enum includes:

| Feature | Code | Description |
|---------|------|-------------|
| `COLUMNS_132` | 1 | 132-column mode |
| `PRINTER` | 2 | Printer port |
| `REGIS_GRAPHICS` | 3 | ReGIS graphics |
| `SIXEL` | 4 | Sixel graphics |
| `SELECTIVE_ERASE` | 6 | Selective erase |
| `DRCS` | 7 | Soft character set |
| `USER_DEFINED_KEYS` | 8 | User-defined keys |
| `NATIONAL_CHARSETS` | 9 | National character sets |
| `LOCATOR` | 16 | DEC locator (mouse) |
| `ANSI_COLOR` | 22 | ANSI color support |
| `RECTANGULAR_EDITING` | 28 | Rectangular editing |
| `ANSI_TEXT_LOCATOR` | 29 | ANSI text locator (mouse) |

## Image Protocol Detection

Detect the terminal's image protocol support using DA1 attributes:

```java
// Query-based detection (most accurate)
ImageProtocol protocol = connection.queryImageProtocol(500);

switch (protocol) {
    case KITTY:
        // Use Kitty graphics protocol
        break;
    case ITERM2:
        // Use iTerm2 inline images
        break;
    case SIXEL:
        // Use Sixel graphics
        break;
    case NONE:
        // No image support detected
        break;
}
```

For faster (but less accurate) detection without querying:

```java
Device device = connection.device();
ImageProtocol protocol = device.getImageProtocol();
```

## Color Capabilities

Get terminal color information:

```java
// Get color depth from terminfo or environment
ColorDepth depth = connection.getColorDepth();

if (depth.supportsTrueColor()) {
    // Use 24-bit RGB colors
} else if (depth.supports256Colors()) {
    // Use 256-color palette
}

// Query terminal for full color capability (uses synchronous I/O)
// This can be called BEFORE openBlocking/openNonBlocking
TerminalColorCapability cap = connection.queryColorCapability(500);
if (cap != null) {
    TerminalTheme theme = cap.getTheme();
    int[] fg = cap.getForegroundRGB();
    int[] bg = cap.getBackgroundRGB();
    int[] cursor = cap.getCursorRGB();
    Map<Integer, int[]> palette = cap.getPaletteColors();
}

// Or use TerminalColorDetector for comprehensive detection with fallbacks
TerminalColorCapability cap = TerminalColorDetector.detect(connection);
```
