---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Terminal'
weight: 8
---

Æsh Readline provides terminal abstraction through the `Connection` and `Terminal` interfaces.

## TerminalConnection

The standard terminal connection:

```java
import org.aesh.tty.terminal.TerminalConnection;

TerminalConnection connection = new TerminalConnection();
connection.openBlocking();  // Blocking mode
// OR
connection.openNonBlocking();  // Non-blocking mode
```

## Connection Interface

### Key Methods

```java
Connection connection = new TerminalConnection();

// Terminal information
Device device = connection.device();
Size size = connection.size();
Charset inputEncoding = connection.inputEncoding();
Charset outputEncoding = connection.outputEncoding();
boolean supportsAnsi = connection.supportsAnsi();

// Write output
connection.write("Hello, World!\n");

// Handlers
Consumer<int[]> stdinHandler = connection.getStdinHandler();
connection.setStdinHandler(handler -> { /* process input */ });

Consumer<Size> sizeHandler = connection.getSizeHandler();
connection.setSizeHandler(size -> { /* handle resize */ });

Consumer<Signal> signalHandler = connection.getSignalHandler();
connection.setSignalHandler(signal -> { /* handle signals */ });

// Theme change handler (CSI ? 997 DSR notifications)
Consumer<TerminalTheme> themeHandler = connection.getThemeChangeHandler();
connection.setThemeChangeHandler(theme -> { /* handle dark/light switch */ });

// Close
connection.close();
connection.close(exitCode);
```

### Raw Mode

Enter raw mode for character-by-character input:

```java
Attributes previousAttributes = connection.enterRawMode();
// ... do work ...
connection.setAttributes(previousAttributes);
```

## Terminal Attributes

Control terminal behavior:

```java
Attributes attrs = connection.getAttributes();

// Local flags
attrs.setLocalFlag(Attributes.LocalFlag.ICANON, false);  // Disable canonical mode
attrs.setLocalFlag(Attributes.LocalFlag.ECHO, false);     // Disable echo
attrs.setLocalFlag(Attributes.LocalFlag.IEXTEN, false);   // Disable implementation-defined processing

// Input flags
attrs.setInputFlag(Attributes.InputFlag.IXON, false);      // Disable XON/XOFF flow control

// Control characters
attrs.setControlChar(Attributes.ControlChar.VMIN, 1);      // Minimum read
attrs.setControlChar(Attributes.ControlChar.VTIME, 0);    // Timeout deciseconds

connection.setAttributes(attrs);
```

## Terminal Size

```java
Size size = connection.size();
int rows = size.getHeight();
int columns = size.getWidth();
```

Handle resize events:

```java
connection.setSizeHandler(newSize -> {
    System.out.println("Terminal resized to: " + newSize.getWidth() + "x" + newSize.getHeight());
});
```

## TerminalBuilder

Build custom terminal configurations:

```java
import org.aesh.readline.terminal.TerminalBuilder;

Terminal terminal = TerminalBuilder.builder()
        .name("myterminal")
        .nativeSignals(true)
        .streams(System.in, System.out)
        .build();
```

## Signal Handling

Handle terminal signals:

```java
connection.setSignalHandler(signal -> {
    switch (signal) {
        case INT:   // Ctrl+C
            System.out.println("Interrupt received");
            break;
        case QUIT:  // Ctrl+\
            System.out.println("Quit received");
            break;
        case TSTP:  // Ctrl+Z
            System.out.println("Suspend received");
            break;
        case CONT:  // Continue after suspend
            System.out.println("Continue received");
            break;
        case WINCH: // Window change
            System.out.println("Window changed");
            break;
    }
});
```

## Cursor Position

Get current cursor position:

```java
Point position = connection.getCursorPosition();
int row = position.getRow();
int col = position.getColumn();
```

## Device Attributes

Query the terminal for its capabilities using DA1/DA2 escape sequences:

```java
// Query primary device attributes (DA1)
DeviceAttributes da = connection.queryPrimaryDeviceAttributes(500);

if (da != null) {
    // Check device class (conformance level)
    int deviceClass = da.getDeviceClass();  // 62=VT220, 64=VT420, etc.

    // Check feature support
    if (da.supportsSixel()) {
        System.out.println("Terminal supports Sixel graphics");
    }
    if (da.supportsAnsiColor()) {
        System.out.println("Terminal supports ANSI colors");
    }
    if (da.supportsMouse()) {
        System.out.println("Terminal supports mouse/locator");
    }
}

// Query both DA1 and DA2
DeviceAttributes full = connection.queryDeviceAttributes(500);
if (full != null && full.hasDA2()) {
    System.out.println("Terminal type: " + full.getTerminalType().getName());
    System.out.println("Firmware version: " + full.getFirmwareVersion());
}
```

### DeviceAttributes Class

The `DeviceAttributes` class provides:

- **Device class**: Conformance level (1=VT100, 62=VT220, 64=VT420)
- **Features**: Set of supported capabilities (Sixel, ANSI color, mouse, etc.)
- **Terminal type**: From DA2 (VT100, VT220, VT420, xterm)
- **Firmware version**: Version number from DA2

```java
// Check if OSC queries are likely supported based on DA1 features
if (da.likelySupportsOscQueries()) {
    int[] bgColor = connection.queryBackgroundColor(500);
}
```

## Terminal Environment Detection

The [`TerminalEnvironment`](terminal-environment) class provides centralized detection of terminal type and capabilities:

```java
import org.aesh.terminal.Device;
import org.aesh.terminal.utils.TerminalEnvironment;

TerminalEnvironment env = TerminalEnvironment.getInstance();

// Detect terminal type
Device.TerminalType type = env.getTerminalType();
System.out.println("Terminal: " + type.getIdentifier());

// Check for specific terminals
if (env.isKitty()) {
    System.out.println("Running in Kitty");
}
if (env.isJetBrains()) {
    System.out.println("Running in JetBrains IDE");
}

// Check multiplexer status
if (env.isInMultiplexer()) {
    System.out.println("Running in tmux or screen");
}

// Check OSC support
if (env.supportsOscQueries()) {
    System.out.println("OSC queries supported");
}

// Get color depth
ColorDepth depth = env.getDefaultColorDepth();
```

The `Device` interface methods also use `TerminalEnvironment` internally:

```java
Device device = connection.device();

// These use TerminalEnvironment
Device.TerminalType type = device.detectTerminalType();
boolean oscSupported = device.supportsOscQueries();
boolean isMultiplexer = device.isMultiplexer();
```

See [Terminal Environment](terminal-environment) for complete documentation.

## Image Protocol Detection

Detect the terminal's inline image support:

```java
// Heuristic detection (fast, no terminal query)
Device device = connection.device();
ImageProtocol protocol = device.getImageProtocol();

// Query-based detection (accurate, uses DA1)
ImageProtocol protocol = connection.queryImageProtocol(500);

switch (protocol) {
    case KITTY:
        // Kitty, Ghostty, Konsole
        break;
    case ITERM2:
        // iTerm2, WezTerm, VS Code, Mintty
        break;
    case SIXEL:
        // xterm, mlterm, foot, contour
        break;
    case NONE:
        // No inline image support
        break;
}
```

The `ImageProtocolDetector` class combines multiple detection methods:
1. Environment variables (KITTY_WINDOW_ID, ITERM_SESSION_ID, etc.)
2. Terminal type string (from TERM environment variable)
3. DA1 device attributes (authoritative Sixel detection)

## OSC Queries

Query the terminal using OSC (Operating System Command) sequences:

```java
// Query background color
int[] bgColor = connection.queryBackgroundColor(500);
if (bgColor != null) {
    int r = bgColor[0], g = bgColor[1], b = bgColor[2];
    boolean isDark = (r + g + b) / 3 < 128;
}

// Query foreground color
int[] fgColor = connection.queryForegroundColor(500);

// Generic OSC query with custom parser
String result = connection.queryOsc(oscCode, "?", 500, input -> {
    // Parse the response
    return parseResponse(input);
});
```

### OSC Support Detection

```java
// Check if OSC queries are supported
if (connection.supportsOscQueries()) {
    // Safe to use OSC queries
}

// More accurate check using DA1 attributes
DeviceAttributes da = connection.queryPrimaryDeviceAttributes(500);
if (connection.supportsOscQueries(da)) {
    // DA1 indicates modern terminal features
}
```

## Theme Mode Detection

Modern terminals can report whether they are using a dark or light color scheme using the `CSI ? 996 n` protocol. This is faster and simpler than parsing OSC 10/11 RGB responses.

```java
// One-shot query
if (connection.supportsThemeQuery()) {
    TerminalTheme theme = connection.queryThemeMode(500);
    // DARK, LIGHT, or null
}

// Subscribe to real-time dark/light mode changes
connection.enableThemeChangeNotification(theme -> {
    System.out.println("Theme changed to: " + theme);
});

// Unsubscribe
connection.disableThemeChangeNotification();
```

Supported terminals include Kitty (0.38.1+), Ghostty (1.0.0+), Contour (0.4.0+), Foot, VTE/GNOME Terminal (0.82.0+), and tmux.

See [Color Detection](color-detection#1-theme-mode-query-csi--996-n--fastest--simplest) for protocol details and [Connection](connection#theme-mode-queries) for the full API reference.
