---
date: '2026-01-26T14:00:00+01:00'
draft: false
title: 'Terminal Colors'
weight: 10
---

The `TerminalColor` class provides comprehensive support for terminal text coloring, including basic ANSI colors, 256-color palette, true color (24-bit RGB), and theme-aware color selection.

## Overview

`TerminalColor` represents a foreground/background color combination for terminal output. It supports three color modes:

| Mode | Colors | Escape Sequence | Example |
|------|--------|-----------------|---------|
| **Basic** | 8/16 | `ESC[30-37m` / `ESC[90-97m` | `new TerminalColor(Color.RED, Color.DEFAULT)` |
| **256-color** | 256 | `ESC[38;5;Nm` | `new TerminalColor(208, 0)` |
| **True color** | 16M | `ESC[38;2;R;G;Bm` | `TerminalColor.fromRGB(255, 87, 51)` |

## Basic Colors

Use the `Color` enum for standard ANSI colors:

```java
import org.aesh.readline.terminal.formatting.Color;
import org.aesh.readline.terminal.formatting.TerminalColor;

// Simple foreground and background
TerminalColor redOnBlack = new TerminalColor(Color.RED, Color.BLACK);
TerminalColor blueOnDefault = new TerminalColor(Color.BLUE, Color.DEFAULT);

// With intensity (NORMAL or BRIGHT)
TerminalColor brightGreen = new TerminalColor(
    Color.GREEN, Color.DEFAULT, Color.Intensity.BRIGHT);
```

### Available Colors

The `Color` enum provides:

| Color | Normal (30-37) | Bright (90-97) |
|-------|----------------|----------------|
| `BLACK` | Dark black | Gray |
| `RED` | Dark red | Bright red |
| `GREEN` | Dark green | Bright green |
| `YELLOW` | Dark yellow/brown | Bright yellow |
| `BLUE` | Dark blue | Bright blue |
| `MAGENTA` | Dark magenta | Bright magenta |
| `CYAN` | Dark cyan | Bright cyan |
| `WHITE` | Light gray | Bright white |
| `DEFAULT` | Terminal default | - |

## 256-Color Palette

Use integer indices (0-255) for the extended color palette:

```java
// Orange text on dark blue background
TerminalColor palette256 = new TerminalColor(208, 17);

// 256-color text with default background
TerminalColor textOnly = new TerminalColor(226, Color.DEFAULT);
```

### Palette Layout

| Range | Description |
|-------|-------------|
| 0-7 | Standard colors (same as basic) |
| 8-15 | High-intensity colors (same as bright) |
| 16-231 | 6x6x6 color cube: `16 + 36*r + 6*g + b` (0 <= r,g,b <= 5) |
| 232-255 | Grayscale: 24 shades from black to white |

## True Color (24-bit RGB)

For terminals that support true color, use RGB values directly:

### From RGB Values

```java
// Foreground only (orange)
TerminalColor orange = TerminalColor.fromRGB(255, 87, 51);

// Both foreground and background
TerminalColor custom = TerminalColor.fromRGB(
    255, 87, 51,    // Foreground: orange
    30, 30, 30      // Background: dark gray
);
```

### From Hex Strings

```java
// Foreground only
TerminalColor coral = TerminalColor.fromHex("#FF5733");
TerminalColor coral2 = TerminalColor.fromHex("FF5733");  // # is optional

// Short form (3 chars expanded to 6)
TerminalColor red = TerminalColor.fromHex("#F00");  // Same as #FF0000

// Both foreground and background
TerminalColor branded = TerminalColor.fromHex("#FF5733", "#1A1A1A");
```

### Checking RGB Mode

```java
TerminalColor color = TerminalColor.fromHex("#FF5733");

if (color.isTrueColor()) {
    int[] fg = color.getTextRGB();       // [255, 87, 51]
    int[] bg = color.getBackgroundRGB(); // null if not set
}
```

## Theme-Aware Colors

The semantic factory methods automatically choose appropriate colors based on the terminal's detected theme (light or dark background).

### Semantic Color Methods

```java
import org.aesh.readline.terminal.TerminalColorDetector;
import org.aesh.terminal.utils.TerminalColorCapability;

// Detect terminal capabilities
TerminalColorCapability cap = TerminalColorDetector.detect(connection);

// Get theme-appropriate colors
TerminalColor error = TerminalColor.forError(cap);      // Red
TerminalColor success = TerminalColor.forSuccess(cap);  // Green
TerminalColor warning = TerminalColor.forWarning(cap);  // Yellow
TerminalColor info = TerminalColor.forInfo(cap);        // Cyan/Blue
TerminalColor highlight = TerminalColor.forHighlight(cap);  // Emphasized
TerminalColor muted = TerminalColor.forMuted(cap);      // Dimmed
```

### How Theme Adaptation Works

| Method | Dark Theme | Light Theme |
|--------|------------|-------------|
| `forError()` | Bright red | Normal red |
| `forSuccess()` | Bright green | Normal green |
| `forWarning()` | Bright yellow | Normal yellow |
| `forInfo()` | Bright cyan | Normal blue |
| `forHighlight()` | Bright white | Black |
| `forMuted()` | Normal white | Normal black |

These methods ensure readable colors on any background. On dark themes, bright colors stand out. On light themes, normal intensity prevents eye strain.

### Example: Status Messages

```java
TerminalColorCapability cap = TerminalColorDetector.detectCached(connection);

public void logError(String message) {
    TerminalString str = new TerminalString(
        "[ERROR] " + message, 
        TerminalColor.forError(cap)
    );
    connection.write(str.toString() + "\n");
}

public void logSuccess(String message) {
    TerminalString str = new TerminalString(
        "[OK] " + message,
        TerminalColor.forSuccess(cap)
    );
    connection.write(str.toString() + "\n");
}
```

## Color Depth Adaptation

Not all terminals support true color. Use `forCapability()` to automatically downgrade colors:

```java
// Create an RGB color
TerminalColor brandOrange = TerminalColor.fromHex("#FF5733");

// Adapt to terminal capabilities
TerminalColor adapted = brandOrange.forCapability(cap);
```

The `forCapability()` method returns:
- The **original RGB color** if the terminal supports true color
- The **nearest 256-color palette index** if only 256 colors are supported
- The **nearest basic 16 color** if only basic colors are supported

### Manual Conversion

You can also convert colors explicitly:

```java
TerminalColor rgb = TerminalColor.fromRGB(255, 128, 0);

// Convert to 256-color (uses 6x6x6 cube or grayscale)
TerminalColor color256 = rgb.toColor256();

// Convert to basic 16 colors
TerminalColor color16 = rgb.toColor16();
```

### Conversion Algorithm

The RGB to 256-color conversion:
1. **Grayscale detection**: If R=G=B, uses grayscale range (232-255)
2. **Color cube mapping**: Maps RGB to nearest point in 6x6x6 cube (16-231)

The RGB to 16-color conversion:
1. **Saturation check**: Low saturation maps to black/white
2. **Dominant channel**: Determines base color from strongest RGB channel
3. **Intensity**: Bright if luminance > 60%, normal otherwise

## Using with TerminalString

`TerminalColor` is typically used with `TerminalString` for styled output:

```java
import org.aesh.readline.terminal.formatting.TerminalString;
import org.aesh.readline.terminal.formatting.TerminalTextStyle;

// Color only
TerminalString colored = new TerminalString("Colored text",
    new TerminalColor(Color.CYAN, Color.DEFAULT));

// Color and style
TerminalString styled = new TerminalString("Bold red",
    new TerminalColor(Color.RED, Color.DEFAULT),
    new TerminalTextStyle(CharacterType.BOLD));

// Write to terminal
connection.write(colored.toString());
```

## Using with ANSIBuilder

For more complex formatting, use `ANSIBuilder`:

```java
import org.aesh.readline.util.ANSIBuilder;

String output = ANSIBuilder.builder()
    .red().text("Error: ").reset()
    .text("Something went wrong")
    .toString();

// With RGB (where supported)
String rgbOutput = ANSIBuilder.builder()
    .trueColor(255, 87, 51).text("Orange text").reset()
    .toString();
```

## Complete Example

```java
import org.aesh.readline.Readline;
import org.aesh.readline.ReadlineBuilder;
import org.aesh.readline.terminal.TerminalColorDetector;
import org.aesh.readline.terminal.formatting.*;
import org.aesh.readline.tty.terminal.TerminalConnection;
import org.aesh.terminal.utils.TerminalColorCapability;

public class ColorfulApp {
    public static void main(String[] args) throws Exception {
        TerminalConnection connection = new TerminalConnection();
        
        // Detect terminal capabilities
        TerminalColorCapability cap = TerminalColorDetector.detectCached(connection);
        
        // Create theme-aware colors
        TerminalColor titleColor = TerminalColor.forHighlight(cap);
        TerminalColor errorColor = TerminalColor.forError(cap);
        TerminalColor successColor = TerminalColor.forSuccess(cap);
        TerminalColor infoColor = TerminalColor.forInfo(cap);
        
        // Create an RGB brand color, adapted to terminal capability
        TerminalColor brandColor = TerminalColor.fromHex("#FF5733")
            .forCapability(cap);
        
        // Display colored output
        connection.write(new TerminalString(
            "=== My Application ===\n", titleColor).toString());
        
        connection.write(new TerminalString(
            "[INFO] ", infoColor).toString() + "Starting up...\n");
        
        connection.write(new TerminalString(
            "[OK] ", successColor).toString() + "Initialization complete\n");
        
        connection.write(new TerminalString(
            "Brand: ", brandColor).toString() + "Custom colored text\n");
        
        // Show detection results
        connection.write("\nDetected: " + cap.getTheme() + " theme, " +
            cap.getColorDepth() + "\n");
        
        connection.close();
    }
}
```

## API Reference

### Constructors

| Constructor | Description |
|-------------|-------------|
| `TerminalColor()` | Default colors |
| `TerminalColor(Color text, Color bg)` | Basic colors |
| `TerminalColor(Color text, Color bg, Intensity)` | Basic colors with intensity |
| `TerminalColor(int text, int bg)` | 256-color palette |
| `TerminalColor(int text, Color bg)` | Mixed: 256-color text, basic background |
| `TerminalColor(Color text, int bg)` | Mixed: basic text, 256-color background |

### Static Factory Methods

| Method | Description |
|--------|-------------|
| `fromRGB(r, g, b)` | Create with RGB foreground |
| `fromRGB(tr, tg, tb, br, bg, bb)` | Create with RGB foreground and background |
| `fromHex(hex)` | Create from hex string (e.g., "#FF5733") |
| `fromHex(textHex, bgHex)` | Create both colors from hex |
| `forError(cap)` | Theme-aware red for errors |
| `forSuccess(cap)` | Theme-aware green for success |
| `forWarning(cap)` | Theme-aware yellow for warnings |
| `forInfo(cap)` | Theme-aware cyan/blue for info |
| `forHighlight(cap)` | Theme-aware emphasis color |
| `forMuted(cap)` | Theme-aware secondary text color |

### Instance Methods

| Method | Description |
|--------|-------------|
| `isTrueColor()` | Check if using RGB mode |
| `getTextRGB()` | Get foreground RGB array or null |
| `getBackgroundRGB()` | Get background RGB array or null |
| `forCapability(cap)` | Adapt to terminal's color depth |
| `toColor256()` | Convert RGB to 256-color palette |
| `toColor16()` | Convert RGB to basic 16 colors |
| `fullString()` | Get complete ANSI escape sequence |
| `toString()` | Get color codes (without ESC prefix) |

## See Also

- [Color Detection](color-detection) - Detecting terminal theme and color depth
- [Examples](examples) - Working code samples
