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

### OSC Support Detection

Check if the terminal supports OSC queries:

```java
// Basic heuristic check
boolean supportsOsc = connection.supportsOscQueries();

// More accurate check using Device Attributes
DeviceAttributes attrs = connection.queryPrimaryDeviceAttributes(500);
boolean supportsOsc = connection.supportsOscQueries(attrs);

// Query-based check (sends DA1 query)
boolean supportsOsc = connection.querySupportsOscQueries(500);
```

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

// Get full color capability info
TerminalColorCapability cap = connection.getColorCapability();
TerminalTheme theme = cap.getTheme();
```
