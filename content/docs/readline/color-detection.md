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
TerminalColorCapability cap = TerminalColorDetector.detect(connection.terminal());

System.out.println("Theme: " + cap.getTheme());
System.out.println("Color depth: " + cap.getColorDepth());
```

### Cached Detection

For repeated access without re-detection overhead:

```java
// Detects once and caches for 5 minutes
TerminalColorCapability cap = TerminalColorDetector.detectCached(connection.terminal());
```

## TerminalColorCapability

The `TerminalColorCapability` class encapsulates all detected information:

```java
TerminalColorCapability cap = TerminalColorDetector.detect(connection.terminal());

// Theme detection
TerminalTheme theme = cap.getTheme();        // DARK, LIGHT, or UNKNOWN
boolean isDark = cap.getTheme().isDark();    // true for DARK or UNKNOWN

// Color depth
ColorDepth depth = cap.getColorDepth();
boolean has256 = depth.supports256Colors();
boolean hasTrueColor = depth.supportsTrueColor();

// Actual RGB colors (may be null if not detectable)
int[] fgRGB = cap.getForegroundRGB();     // [r, g, b] or null
int[] bgRGB = cap.getBackgroundRGB();     // [r, g, b] or null
int[] cursorRGB = cap.getCursorRGB();     // [r, g, b] or null

// Palette colors (ANSI 16 colors, indices 0-15)
if (cap.hasPaletteColors()) {
    Map<Integer, int[]> palette = cap.getPaletteColors();
    int[] red = cap.getPaletteColor(1);   // Standard red
    int[] brightRed = cap.getPaletteColor(9);  // Bright red
}
```

The `detect()` method queries (in priority order):
- Theme mode (CSI ? 996 n) - if supported by the terminal
- Foreground color (OSC 10)
- Background color (OSC 11)
- Cursor color (OSC 12) - if supported
- ANSI 16 palette colors (OSC 4, indices 0-15) - if supported

### Suggested Color Codes

Get ANSI color codes that work well with the detected theme:

```java
TerminalColorCapability cap = TerminalColorDetector.detect(connection.terminal());

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
TerminalColorCapability detected = TerminalColorDetector.detect(connection.terminal());
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

The detector uses multiple strategies to determine the terminal theme, in priority order:

### 1. Theme Mode Query (CSI ? 996 n) — Fastest & Simplest

The preferred method on supporting terminals. Sends a single escape sequence and gets back a direct dark/light answer — no RGB parsing or luminance calculation needed.

```
CSI ? 996 n  →  CSI ? 997 ; 1 n  (dark)
                CSI ? 997 ; 2 n  (light)
```

This is based on the [Contour VT extension](https://contour-terminal.org/vt-extensions/color-palette-update-notifications/) for color palette update notifications.

#### Using via Connection

```java
// One-shot query: returns DARK, LIGHT, or null (unsupported/timeout)
TerminalTheme theme = connection.terminal().queryThemeMode(500);

if (theme != null) {
    System.out.println("Theme: " + theme);
} else {
    // Fall back to OSC 10/11 or environment detection
}
```

#### Checking Support

```java
// Check before querying (avoids timeout on unsupported terminals)
if (connection.terminal().supportsThemeQuery()) {
    TerminalTheme theme = connection.terminal().queryThemeMode(500);
}
```

#### Supported Terminals

| Terminal | Version | Notes |
|----------|---------|-------|
| Contour | 0.4.0+ | Origin of the protocol |
| Ghostty | 1.0.0+ | Full support |
| Kitty | 0.38.1+ | Full support |
| tmux | — | Passes through to underlying terminal |
| VTE / GNOME Terminal | 0.82.0+ | Full support |
| Foot | — | Full support |

The `TerminalColorDetector` tries this method first when the terminal supports it, then falls back to OSC queries and environment detection.

### 2. OSC Color Queries (Most Accurate Fallback)

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
int[] bg = connection.terminal().queryBackgroundColor(500);
if (bg != null) {
    int r = bg[0], g = bg[1], b = bg[2];
    boolean isDark = (r + g + b) / 3 < 128;
    System.out.println("Background: RGB(" + r + "," + g + "," + b + ")");
}

// Query foreground color (OSC 10)
int[] fg = connection.terminal().queryForegroundColor(500);

// Query cursor color (OSC 12)
int[] cursor = connection.terminal().queryCursorColor(500);

// Generic OSC query for any code
String result = connection.terminal().queryOsc(oscCode, "?", 500, responseParser);
```

#### Batch Color Queries

For better performance when querying multiple colors, use batch queries. This reduces latency from O(n × timeout) to O(timeout) by sending all queries at once:

```java
import org.aesh.terminal.tty.TerminalColorDetector;
import org.aesh.terminal.utils.ANSI;

// Query foreground, background, and cursor in one operation (~50-100ms vs ~600ms)
Map<Integer, int[]> colors = TerminalColorDetector.queryColors(connection.terminal(), 500);

int[] fg = colors.get(ANSI.OSC_FOREGROUND);   // OSC 10
int[] bg = colors.get(ANSI.OSC_BACKGROUND);   // OSC 11
int[] cursor = colors.get(ANSI.OSC_CURSOR_COLOR);  // OSC 12

// Query multiple palette colors at once
Map<Integer, int[]> palette = TerminalColorDetector.queryPaletteColors(
    connection.terminal(), 500, 0, 1, 2, 3, 4, 5, 6, 7);

// Query all 16 ANSI colors
Map<Integer, int[]> ansi16 = TerminalColorDetector.queryAnsi16Colors(connection.terminal(), 500);
```

#### Fallback When OSC Not Supported

Not all terminals support OSC queries. Use `queryColorsWithFallback()` for graceful degradation:

```java
// Always returns colors - actual or estimated based on environment
Map<Integer, int[]> colors = TerminalColorDetector.queryColorsWithFallback(connection.terminal(), 500);

int[] bg = colors.get(ANSI.OSC_BACKGROUND);
// bg is never null - will be estimated if OSC queries failed

// Check if it's a dark theme
boolean isDark = TerminalColorDetector.isDarkColor(bg);
```

You can also check support before querying:

```java
// Check if OSC queries are supported
if (TerminalColorDetector.isOscColorQuerySupported(connection.terminal())) {
    // OSC queries will work
    Map<Integer, int[]> colors = TerminalColorDetector.queryColors(connection.terminal(), 500);
} else {
    // Use environment-based detection
    TerminalTheme theme = TerminalColorDetector.detectThemeFromEnvironment();
}

// Check palette query support specifically
if (connection.terminal().supportsPaletteQuery()) {
    Map<Integer, int[]> palette = TerminalColorDetector.queryAnsi16Colors(connection.terminal(), 500);
}
```

#### OSC Support Detection with Device Attributes

Use DA1 device attributes to determine if OSC queries are likely to work:

```java
// Query device attributes first
DeviceAttributes da = connection.terminal().queryPrimaryDeviceAttributes(500);

// Check if OSC queries are likely supported
if (connection.terminal().supportsOscQueries(da)) {
    int[] bgColor = connection.terminal().queryBackgroundColor(500);
    // ...
}

// Or use combined query-based check
if (connection.terminal().querySupportsOscQueries(500)) {
    // Terminal likely supports OSC queries
}
```

Terminals that report modern features (ANSI color, Sixel graphics, device class >= 62) typically support OSC queries.

#### Converting RGB to ANSI Color Codes

After querying RGB colors, you can convert them to ANSI color codes using the `ANSI` utility class:

```java
int[] bgColor = connection.terminal().queryBackgroundColor(500);

if (bgColor != null) {
    // Convert to 256-color palette index
    int paletteIndex = ANSI.rgbTo256Color(bgColor[0], bgColor[1], bgColor[2]);

    // Convert to basic ANSI foreground code (30-37 or 90-97)
    int ansiCode = ANSI.rgbToAnsiColor(bgColor[0], bgColor[1], bgColor[2]);

    // Check brightness for theme detection
    boolean isDark = !ANSI.rgbIsBright(bgColor[0], bgColor[1], bgColor[2]);

    // Reverse conversion: get RGB from palette index
    int[] rgb = ANSI.color256ToRgb(paletteIndex);
}
```

See [Terminal Colors](terminal-colors#ansi-color-utilities) for complete documentation of RGB/ANSI conversion utilities.

### 3. Environment Variables

Checks standard environment variables via [`TerminalEnvironment`](terminal-environment):

| Variable | Example | Meaning |
|----------|---------|---------|
| `COLORFGBG` | `15;0` | Foreground;Background color indices |
| `COLORTERM` | `truecolor` | Color depth hint |
| `APPLE_INTERFACE_STYLE` | `Dark` | macOS dark mode |
| `TERM_PROGRAM` | `iTerm.app` | Terminal program identifier |
| `KITTY_WINDOW_ID` | `1` | Kitty terminal indicator |
| `ITERM_SESSION_ID` | `w0t0p0` | iTerm2 session indicator |
| `WEZTERM_PANE` | `0` | WezTerm terminal indicator |
| `GHOSTTY_RESOURCES_DIR` | `/path` | Ghostty terminal indicator |

The `TerminalEnvironment` class parses these once and caches the results. See [Terminal Environment](terminal-environment) for details on all supported environment variables and terminal types.

### 4. Terminal-Specific Detection

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

When running inside tmux or GNU Screen, color detection requires special handling. The [`TerminalEnvironment`](terminal-environment) class provides centralized multiplexer detection:

```java
import org.aesh.terminal.utils.TerminalEnvironment;

TerminalEnvironment env = TerminalEnvironment.getInstance();

// Check if running in a multiplexer
if (env.isInTmux()) {
    System.out.println("Running inside tmux");
}

if (env.isInMultiplexer()) {
    System.out.println("Running inside tmux or screen");
}

// Check if passthrough is enabled
if (env.isTmuxPassthroughEnabled()) {
    System.out.println("OSC passthrough is enabled");
}

// Legacy static methods still work
if (TerminalColorDetector.isRunningInTmux()) {
    System.out.println("Running inside tmux");
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

## Real-Time Theme Change Notifications

Terminals that support the CSI ? 996 n protocol can also send **unsolicited** notifications when the user switches between dark and light mode. This allows your application to adapt its colors in real time without polling.

### How It Works

1. Your application sends `CSI ? 2031 h` to **subscribe** to theme change notifications
2. The terminal sends `CSI ? 997 ; 1 n` (dark) or `CSI ? 997 ; 2 n` (light) whenever the theme changes
3. Your application sends `CSI ? 2031 l` to **unsubscribe**

The `EventDecoder` in the input pipeline automatically intercepts these unsolicited DSR sequences and routes them to a registered handler, preventing them from appearing as garbage in the readline buffer.

### Subscribing to Theme Changes

```java
import org.aesh.terminal.utils.TerminalTheme;

// One-call setup: register handler and enable notifications
connection.terminal().enableThemeChangeNotification(theme -> {
    System.out.println("Theme changed to: " + theme);
    // Update your cached color capability
    capability = new TerminalColorCapability(capability.getColorDepth(), theme);
});
```

Or separately:

```java
// Register handler first
connection.setThemeChangeHandler(theme -> {
    System.out.println("Theme changed to: " + theme);
});

// Then enable notifications
connection.terminal().enableThemeChangeNotification();
```

### Unsubscribing

```java
// Stop the terminal from sending notifications
connection.terminal().disableThemeChangeNotification();

// Optionally remove the handler
connection.setThemeChangeHandler(null);
```

### Performance

When no theme change handler is registered, the DSR filtering in `EventDecoder` has **zero overhead** — a single null check on the fast path. When a handler is registered but the input contains no ESC bytes, the overhead is a single linear scan with no allocation. The state machine only activates when an ESC byte is found in the input.

### Supported Terminals

Theme change notifications use the same terminal support as the CSI ? 996 n query. See the [Theme Mode Query](#1-theme-mode-query-csi--996-n--fastest--simplest) section above for the full compatibility table.

See the [Connection](connection#theme-mode-queries) documentation for the complete API reference.

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
        TerminalColorCapability cap = TerminalColorDetector.detect(connection.terminal());
        
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
        colors = TerminalColorDetector.detectCached(conn.terminal());
        // Use 'colors' throughout the application
    }
}
```

## Supported Terminals

The following terminals have been tested with full detection support:

| Terminal | OSC Query | Theme DSR | Config File | Notes |
|----------|-----------|-----------|-------------|-------|
| iTerm2 | Yes | No | - | Full OSC support |
| Kitty | Yes | Yes (0.38.1+) | - | Full support |
| WezTerm | Yes | No | - | Full OSC support |
| Alacritty | Yes | No | Yes | Both methods work |
| Ghostty | Yes | Yes (1.0.0+) | - | Full support |
| GNOME Terminal | Yes | Yes (VTE 0.82.0+) | - | Full support |
| Konsole | Yes | No | - | Full OSC support |
| Windows Terminal | Yes | No | Yes | Both methods work |
| Foot | Limited | Yes | - | Theme DSR preferred |
| Contour | Yes | Yes (0.4.0+) | - | Origin of Theme DSR protocol |
| VS Code Terminal | Limited | No | Yes | Config file preferred |
| JetBrains IDEs | No | No | Yes | Config file only |
| tmux | Yes* | Yes* | - | *Requires 3.3+ or passthrough |
| ConEmu/Cmder | No | No | Yes | Config file only |
| Apple Terminal | Limited | No | - | Basic support |

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
TerminalColorCapability cap = TerminalColorDetector.detect(connection.terminal(), 100);
```

## Using TerminalColor with Detection

The `TerminalColor` class integrates seamlessly with color detection, providing theme-aware factory methods and automatic color depth adaptation.

### Theme-Aware Semantic Colors

Use semantic factory methods that automatically adjust for the detected terminal theme:

```java
import org.aesh.readline.terminal.formatting.TerminalColor;
import org.aesh.terminal.utils.TerminalColorCapability;

TerminalColorCapability cap = TerminalColorDetector.detect(connection.terminal());

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
TerminalColorCapability cap = TerminalColorDetector.detectCached(connection.terminal());

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
TerminalColorCapability cap = TerminalColorDetector.detect(connection.terminal());
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

## Terminal Mode Detection

In addition to colors, the `TerminalCapabilities` class detects support for DEC private modes. These probes are included in the same batched query as the color detection — zero additional startup cost.

### Detected Modes

| Mode | Method | Description |
|---|---|---|
| Mode 2026 | `synchronizedOutputSupport()` | Synchronized output (BSU/ESU) — prevents screen tearing |
| Mode 2027 | `graphemeClusterSupport()` | Grapheme cluster segmentation — correct emoji width |

### Three-State Detection

Each mode returns a `ModeSupport` enum with three states:

| State | Meaning |
|---|---|
| `SUPPORTED` | Terminal responded positively to the DECRQM probe |
| `NOT_SUPPORTED` | Terminal responded to DA1 (speaks escape sequences) but does not support this mode |
| `NO_RESPONSE` | Terminal did not respond at all (dumb terminal — no further probes safe) |

```java
TerminalCapabilities caps = TerminalCapabilities.detectAsync();
caps.awaitColors(500, TimeUnit.MILLISECONDS);

ModeSupport sync = caps.synchronizedOutputSupport();
if (sync == ModeSupport.SUPPORTED) {
    // Safe to use Mode 2026 for flicker-free updates
}

ModeSupport grapheme = caps.graphemeClusterSupport();
if (grapheme == ModeSupport.SUPPORTED) {
    // Terminal handles emoji width correctly
}
```

### Native Grapheme Clustering

Some terminals natively group grapheme clusters (correct emoji width) without supporting Mode 2027. The `detectFull()` method includes a cursor-position probe that detects this:

```java
TerminalCapabilities caps = TerminalCapabilities.detectFull();
Boolean nativeGC = caps.nativeGraphemeClustering();
if (nativeGC != null && nativeGC) {
    // Terminal groups emoji correctly even without Mode 2027
}
```

The cursor probe writes a test emoji, measures the cursor advance, and cleans up. It only runs when DA1 responded but Mode 2027 is not supported — never on dumb terminals.
