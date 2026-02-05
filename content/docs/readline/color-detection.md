---
date: '2026-01-26T12:00:00+01:00'
draft: false
title: 'Color Detection'
weight: 9
---

Æsh Readline provides automatic terminal color detection to help your application adapt its color scheme to the user's terminal environment.

## Overview

The `TerminalColorDetector` API detects:

- **Color Depth** - How many colors the terminal supports (8, 16, 256, or true color)
- **Theme** - Whether the terminal has a light or dark background
- **RGB Colors** - The actual foreground and background colors (when available)

This allows your application to automatically choose appropriate colors that will be readable on any terminal background.

## Quick Start

### Fast Detection (Environment Only)

For immediate, non-blocking detection based on environment variables:

```java
import org.aesh.terminal.utils.TerminalColorCapability;

TerminalColorCapability cap = TerminalColorCapability.detectFromEnvironment();

if (cap.getTheme().isDark()) {
    // Use light text colors
} else {
    // Use dark text colors
}
```

### Full Detection (With Terminal Query)

For more accurate detection that queries the terminal:

```java
import org.aesh.readline.terminal.TerminalColorDetector;
import org.aesh.readline.tty.terminal.TerminalConnection;

TerminalConnection connection = new TerminalConnection();
TerminalColorCapability cap = TerminalColorDetector.detect(connection);

System.out.println("Theme: " + cap.getTheme());
System.out.println("Color depth: " + cap.getColorDepth());
```

### Cached Detection

For repeated access without re-detection overhead:

```java
// Detects once and caches for 5 minutes
TerminalColorCapability cap = TerminalColorDetector.detectCached(connection);
```

## TerminalColorCapability

The `TerminalColorCapability` class encapsulates all detected information:

```java
TerminalColorCapability cap = TerminalColorDetector.detect(connection);

// Theme detection
TerminalTheme theme = cap.getTheme();        // DARK, LIGHT, or UNKNOWN
boolean isDark = cap.getTheme().isDark();    // true for DARK or UNKNOWN

// Color depth
ColorDepth depth = cap.getColorDepth();
boolean has256 = depth.supports256Colors();
boolean hasTrueColor = depth.supportsTrueColor();

// Actual RGB colors (may be null if not detectable)
int[] bgRGB = cap.getBackgroundRGB();  // [r, g, b] or null
int[] fgRGB = cap.getForegroundRGB();  // [r, g, b] or null
```

### Suggested Color Codes

Get ANSI color codes that work well with the detected theme:

```java
TerminalColorCapability cap = TerminalColorDetector.detect(connection);

// These return appropriate codes for the detected background
int normalText = cap.getSuggestedForegroundCode();  // 30 (black) or 37 (white)
int errorText = cap.getSuggestedErrorCode();        // 31 or 91 (red variants)
int successText = cap.getSuggestedSuccessCode();    // 32 or 92 (green variants)
int warningText = cap.getSuggestedWarningCode();    // 33 or 93 (yellow variants)
int infoText = cap.getSuggestedInfoCode();          // 34 or 94 (blue variants)
int debugText = cap.getSuggestedDebugCode();        // 90 or 37 (gray/white variants)
int traceText = cap.getSuggestedTraceCode();        // 90 (gray - least prominent)
int timestampText = cap.getSuggestedTimestampCode(); // 36 or 96 (cyan variants)
int messageText = cap.getSuggestedMessageCode();     // 35 or 95 (magenta variants)

// Use in ANSI escape sequences
connection.write("\u001B[" + errorText + "mError: Something went wrong\u001B[0m\n");
```

### All Suggested Colors

| Method | Dark Theme | Light Theme | Use Case |
|--------|------------|-------------|----------|
| `getSuggestedForegroundCode()` | 37 (white) | 30 (black) | Normal text |
| `getSuggestedErrorCode()` | 91 (bright red) | 31 (red) | Error messages |
| `getSuggestedSuccessCode()` | 92 (bright green) | 32 (green) | Success messages |
| `getSuggestedWarningCode()` | 93 (bright yellow) | 33 (yellow) | Warnings |
| `getSuggestedInfoCode()` | 94 (bright blue) | 34 (blue) | Info messages |
| `getSuggestedDebugCode()` | 37 (white) | 90 (gray) | Debug messages |
| `getSuggestedTraceCode()` | 242 (256-color gray) | 90 (gray) | Trace messages (least prominent) |
| `getSuggestedTimestampCode()` | 96 (bright cyan) | 36 (cyan) | Timestamps in logs |
| `getSuggestedMessageCode()` | 95 (bright magenta) | 35 (magenta) | Highlighted messages |

The log level colors follow a prominence hierarchy from most to least visible:
**ERROR > WARN > INFO > DEBUG > TRACE**

### Customizing Suggested Colors

You can override the default suggested colors using the `Builder`:

```java
// Start with detected capability and customize specific colors
TerminalColorCapability detected = TerminalColorDetector.detect(connection);
TerminalColorCapability custom = TerminalColorCapability.builder(detected)
    .errorCode(196)      // Custom 256-color bright red
    .successCode(46)     // Custom 256-color green
    .timestampCode(244)  // Custom 256-color gray for timestamps
    .build();

// Now getSuggestedErrorCode() returns 196 instead of theme-based default
int error = custom.getSuggestedErrorCode();  // Returns 196
```

Or build from scratch:

```java
TerminalColorCapability custom = TerminalColorCapability.builder()
    .colorDepth(ColorDepth.COLORS_256)
    .theme(TerminalTheme.DARK)
    .errorCode(196)
    .successCode(46)
    .warningCode(208)
    .infoCode(39)
    .debugCode(250)      // 256-color light gray
    .traceCode(240)      // 256-color dark gray
    .timestampCode(244)
    .messageCode(255)
    .foregroundCode(252)
    .build();
```

This is useful when:
- You have specific brand colors to use
- User preferences should override theme-based defaults
- You need 256-color or RGB codes instead of basic ANSI colors

## ColorDepth

The `ColorDepth` enum represents terminal color capabilities:

| Value | Colors | Description |
|-------|--------|-------------|
| `NO_COLOR` | 0 | No color support |
| `COLORS_8` | 8 | Basic ANSI colors (30-37) |
| `COLORS_16` | 16 | Extended ANSI colors (30-37, 90-97) |
| `COLORS_256` | 256 | 256-color palette (38;5;N) |
| `TRUE_COLOR` | 16M | 24-bit RGB (38;2;R;G;B) |

```java
ColorDepth depth = cap.getColorDepth();

if (depth.supportsTrueColor()) {
    // Use full RGB colors
    connection.write("\u001B[38;2;255;128;0mOrange text\u001B[0m");
} else if (depth.supports256Colors()) {
    // Use 256-color palette
    connection.write("\u001B[38;5;208mOrange text\u001B[0m");
} else {
    // Fall back to basic colors
    connection.write("\u001B[33mYellow text\u001B[0m");
}
```

## TerminalTheme

The `TerminalTheme` enum represents the terminal's background brightness:

| Value | Description |
|-------|-------------|
| `DARK` | Dark background (use light text) |
| `LIGHT` | Light background (use dark text) |
| `UNKNOWN` | Could not detect (assumes dark) |

```java
TerminalTheme theme = cap.getTheme();

switch (theme) {
    case DARK:
        // Use bright/light colors for text
        break;
    case LIGHT:
        // Use dark colors for text
        break;
    case UNKNOWN:
        // Default to dark theme assumption
        break;
}

// Helper method - returns true for DARK and UNKNOWN
if (theme.isDark()) {
    // Use light colors
}
```

## Detection Methods

The detector uses multiple strategies to determine the terminal theme:

### 1. OSC Color Queries (Most Accurate)

Queries the terminal directly for its background color using OSC 10/11 escape sequences:

```
ESC ] 11 ; ? BEL  →  ESC ] 11 ; rgb:RRRR/GGGG/BBBB BEL
```

This works with most modern terminal emulators including:
- iTerm2, Kitty, WezTerm, Alacritty, Ghostty
- GNOME Terminal, Konsole, xterm
- Windows Terminal

#### Direct Color Queries via Connection

You can also query colors directly using the `Connection` interface:

```java
// Query background color (OSC 11)
int[] bg = connection.queryBackgroundColor(500);
if (bg != null) {
    int r = bg[0], g = bg[1], b = bg[2];
    boolean isDark = (r + g + b) / 3 < 128;
    System.out.println("Background: RGB(" + r + "," + g + "," + b + ")");
}

// Query foreground color (OSC 10)
int[] fg = connection.queryForegroundColor(500);

// Query cursor color (OSC 12)
int[] cursor = connection.queryCursorColor(500);

// Generic OSC query for any code
String result = connection.queryOsc(oscCode, "?", 500, responseParser);
```

#### OSC Support Detection with Device Attributes

Use DA1 device attributes to determine if OSC queries are likely to work:

```java
// Query device attributes first
DeviceAttributes da = connection.queryPrimaryDeviceAttributes(500);

// Check if OSC queries are likely supported
if (connection.supportsOscQueries(da)) {
    int[] bgColor = connection.queryBackgroundColor(500);
    // ...
}

// Or use combined query-based check
if (connection.querySupportsOscQueries(500)) {
    // Terminal likely supports OSC queries
}
```

Terminals that report modern features (ANSI color, Sixel graphics, device class >= 62) typically support OSC queries.

### 2. Environment Variables

Checks standard environment variables:

| Variable | Example | Meaning |
|----------|---------|---------|
| `COLORFGBG` | `15;0` | Foreground;Background color indices |
| `COLORTERM` | `truecolor` | Color depth hint |
| `APPLE_INTERFACE_STYLE` | `Dark` | macOS dark mode |

### 3. Terminal-Specific Detection

For terminals that don't support OSC queries, the detector reads configuration files:

#### Visual Studio Code
Reads `settings.json` for `workbench.colorTheme`:
- Linux: `~/.config/Code/User/settings.json`
- macOS: `~/Library/Application Support/Code/User/settings.json`
- Windows: `%APPDATA%\Code\User\settings.json`

#### JetBrains IDEs (IntelliJ, PyCharm, etc.)
Reads `colors.scheme.xml` for the color scheme:
- Linux: `~/.config/JetBrains/<Product>/options/colors.scheme.xml`
- macOS: `~/Library/Application Support/JetBrains/<Product>/options/colors.scheme.xml`
- Windows: `%APPDATA%\JetBrains\<Product>\options\colors.scheme.xml`

#### Alacritty
Reads `alacritty.toml` or `alacritty.yml`:
- Linux/macOS: `~/.config/alacritty/alacritty.toml`
- Windows: `%APPDATA%\alacritty\alacritty.toml`

#### Windows Terminal
Reads `settings.json` for `colorScheme`:
- `%LOCALAPPDATA%\Packages\Microsoft.WindowsTerminal_*/LocalState/settings.json`

#### ConEmu/Cmder
Reads `ConEmu.xml` for palette settings.

#### Windows System Dark Mode
Queries the Windows registry for `AppsUseLightTheme`.

## Multiplexer Support (tmux/screen)

When running inside tmux or GNU Screen, color detection requires special handling:

```java
// Check if running in a multiplexer
if (TerminalColorDetector.isRunningInTmux()) {
    System.out.println("Running inside tmux");
}

if (TerminalColorDetector.isRunningInMultiplexer()) {
    System.out.println("Running inside tmux or screen");
}
```

### tmux Passthrough

For tmux versions before 3.3, OSC queries need to be wrapped in DCS passthrough sequences. The detector handles this automatically when:

1. tmux 3.3+ is detected (native OSC support)
2. `allow-passthrough` is enabled in tmux config
3. `TMUX_PASSTHROUGH=1` environment variable is set

To enable passthrough in tmux:

```bash
# In tmux.conf
set -g allow-passthrough on

# Or at runtime
tmux set -g allow-passthrough on
```

## Example Application

Here's a complete example that adapts colors based on detection:

```java
import org.aesh.readline.terminal.TerminalColorDetector;
import org.aesh.readline.tty.terminal.TerminalConnection;
import org.aesh.terminal.utils.TerminalColorCapability;

public class AdaptiveColorApp {
    private static final String RESET = "\u001B[0m";
    
    public static void main(String[] args) throws Exception {
        TerminalConnection connection = new TerminalConnection();
        TerminalColorCapability cap = TerminalColorDetector.detect(connection);
        
        // Get theme-appropriate colors
        int error = cap.getSuggestedErrorCode();
        int success = cap.getSuggestedSuccessCode();
        int info = cap.getSuggestedInfoCode();
        
        // Display messages with appropriate colors
        connection.write("\u001B[" + info + "m[INFO]\u001B[0m Application started\n");
        connection.write("\u001B[" + success + "m[OK]\u001B[0m Configuration loaded\n");
        connection.write("\u001B[" + error + "m[ERROR]\u001B[0m Unable to connect\n");
        
        // Show detection results
        connection.write("\nDetected: " + cap.getTheme() + " theme, " + 
                        cap.getColorDepth() + "\n");
        
        if (cap.hasBackgroundColor()) {
            int[] bg = cap.getBackgroundRGB();
            connection.write("Background: RGB(" + bg[0] + "," + bg[1] + "," + bg[2] + ")\n");
        }
        
        connection.close();
    }
}
```

## Performance Considerations

### Detection Timing

| Method | Time | Blocking |
|--------|------|----------|
| `detectFromEnvironment()` | < 1ms | No |
| `detectFast()` | < 1ms | No |
| `detect()` | 50-200ms | Yes (waits for terminal response) |
| `detectCached()` | < 1ms (after first call) | Only on first call |

### Recommendations

1. **Use `detectCached()`** for most applications - it caches results for 5 minutes
2. **Use `detectFast()`** if you can't wait for terminal queries
3. **Call detection once** at startup, not on every output
4. **Respect user overrides** - allow users to specify theme preference

```java
// Good: Detect once at startup
public class MyApp {
    private static TerminalColorCapability colors;
    
    public static void main(String[] args) {
        TerminalConnection conn = new TerminalConnection();
        colors = TerminalColorDetector.detectCached(conn);
        // Use 'colors' throughout the application
    }
}
```

## Supported Terminals

The following terminals have been tested with full detection support:

| Terminal | OSC Query | Config File | Notes |
|----------|-----------|-------------|-------|
| iTerm2 | Yes | - | Full support |
| Kitty | Yes | - | Full support |
| WezTerm | Yes | - | Full support |
| Alacritty | Yes | Yes | Both methods work |
| Ghostty | Yes | - | Full support |
| GNOME Terminal | Yes | - | Full support |
| Konsole | Yes | - | Full support |
| Windows Terminal | Yes | Yes | Both methods work |
| VS Code Terminal | Limited | Yes | Config file preferred |
| JetBrains IDEs | No | Yes | Config file only |
| tmux | Yes* | - | *Requires 3.3+ or passthrough |
| ConEmu/Cmder | No | Yes | Config file only |
| Apple Terminal | Limited | - | Basic support |

## Troubleshooting

### Theme Not Detected

If theme detection returns `UNKNOWN`:

1. **Check environment variables**: `echo $TERM $COLORTERM`
2. **Verify terminal support**: Not all terminals support OSC queries
3. **tmux users**: Enable passthrough with `set -g allow-passthrough on`
4. **IDE users**: Ensure the IDE config files are in standard locations

### Wrong Theme Detected

If the detected theme doesn't match your terminal:

1. **Custom color schemes**: Some schemes may not be in the known list
2. **Override manually**: Allow users to specify their preference
3. **Report the issue**: Help us improve detection for your terminal

### OSC Query Hangs

If detection seems to hang:

1. **Reduce timeout**: Use `detect(connection, 100)` for shorter timeout
2. **Use fast detection**: `detectFast(connection)` skips OSC queries
3. **Check terminal**: Some terminals don't respond to OSC queries

```java
// With custom timeout (milliseconds)
TerminalColorCapability cap = TerminalColorDetector.detect(connection, 100);
```

## Using TerminalColor with Detection

The `TerminalColor` class integrates seamlessly with color detection, providing theme-aware factory methods and automatic color depth adaptation.

### Theme-Aware Semantic Colors

Use semantic factory methods that automatically adjust for the detected terminal theme:

```java
import org.aesh.readline.terminal.formatting.TerminalColor;
import org.aesh.terminal.utils.TerminalColorCapability;

TerminalColorCapability cap = TerminalColorDetector.detect(connection);

// These methods automatically choose appropriate colors for the theme
TerminalColor error = TerminalColor.forError(cap);          // Red, bright on dark
TerminalColor success = TerminalColor.forSuccess(cap);      // Green, bright on dark
TerminalColor warning = TerminalColor.forWarning(cap);      // Yellow, bright on dark
TerminalColor info = TerminalColor.forInfo(cap);            // Cyan/Blue, bright on dark
TerminalColor highlight = TerminalColor.forHighlight(cap);  // Emphasized text
TerminalColor muted = TerminalColor.forMuted(cap);          // Secondary/dim text
TerminalColor timestamp = TerminalColor.forTimestamp(cap);  // Cyan, for log timestamps
TerminalColor message = TerminalColor.forMessage(cap);      // Magenta, for highlighted messages
```

On **dark themes**, these return bright variants for readability. On **light themes**, they use normal intensity to avoid glaring colors.

### Example: Adaptive Status Messages

```java
TerminalColorCapability cap = TerminalColorDetector.detectCached(connection);

// Create semantic colors once
TerminalColor errorColor = TerminalColor.forError(cap);
TerminalColor successColor = TerminalColor.forSuccess(cap);
TerminalColor infoColor = TerminalColor.forInfo(cap);

// Use with TerminalString
TerminalString errorMsg = new TerminalString("[ERROR] Connection failed", errorColor);
TerminalString okMsg = new TerminalString("[OK] Connected", successColor);
TerminalString infoMsg = new TerminalString("[INFO] Processing...", infoColor);

connection.write(errorMsg.toString() + "\n");
connection.write(okMsg.toString() + "\n");
connection.write(infoMsg.toString() + "\n");
```

### Example: Log Output with Timestamps

Use `ANSIBuilder` for rich log-style output with timestamps:

```java
TerminalColorCapability cap = TerminalColorDetector.detect(connection);
ANSIBuilder builder = ANSIBuilder.builder(cap);

// Log line with timestamp, level, and message
connection.write(builder
    .timestamp("2024-01-15 10:30:45").append(" ")
    .error("[ERROR]").append(" ")
    .append("Connection to database failed")
    .toLine());

builder.reset();
connection.write(builder
    .timestamp("2024-01-15 10:30:46").append(" ")
    .warning("[WARN]").append(" ")
    .append("Low memory condition detected")
    .toLine());

builder.reset();
connection.write(builder
    .timestamp("2024-01-15 10:30:47").append(" ")
    .info("[INFO]").append(" ")
    .message("Application started successfully")
    .toLine());

builder.reset();
connection.write(builder
    .timestamp("2024-01-15 10:30:48").append(" ")
    .debug("[DEBUG]").append(" ")
    .append("Configuration loaded from /etc/app.conf")
    .toLine());

builder.reset();
connection.write(builder
    .timestamp("2024-01-15 10:30:49").append(" ")
    .trace("[TRACE]").append(" ")
    .append("Entering method processRequest()")
    .toLine());
```

Output adapts automatically to the terminal theme - bright colors on dark backgrounds, normal colors on light backgrounds. Debug and trace use subdued colors (white/gray) to be less prominent than the colored log levels.

### Color Depth Adaptation

When using RGB colors on terminals with limited color support, use `forCapability()` to automatically downgrade:

```java
// Create an RGB color
TerminalColor brandColor = TerminalColor.fromHex("#FF5733");

// Adapt to terminal capabilities
TerminalColor adapted = brandColor.forCapability(cap);

// 'adapted' will be:
// - The original RGB if terminal supports true color
// - Nearest 256-color palette index if terminal supports 256 colors
// - Nearest basic 16 color if terminal only supports basic colors
```

This ensures your colors always work, even on limited terminals. See [Terminal Colors](terminal-colors) for more details on `TerminalColor` RGB and formatting features.
