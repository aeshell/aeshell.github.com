---
date: '2026-02-06T14:00:00+01:00'
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

## HSL Color Support

HSL (Hue, Saturation, Lightness) is often more intuitive than RGB for color selection and is especially useful for generating color palettes. The color wheel makes it easy to:

- **Create color variations** by adjusting lightness
- **Generate harmonious colors** by rotating hue
- **Control color intensity** via saturation

### HSL Value Ranges

| Parameter | Range | Description |
|-----------|-------|-------------|
| **Hue** | 0-360 | Degrees on the color wheel |
| **Saturation** | 0-100 | Percentage (0 = gray, 100 = vivid) |
| **Lightness** | 0-100 | Percentage (0 = black, 50 = pure color, 100 = white) |

**Important:** HSL values use **percentages (0-100)**, not decimal fractions (0-1). For example, use `80` for 80% lightness, not `0.8`.

```java
// Correct - using percentages
TerminalColor.fromHSL(120, 80, 65);  // H=120°, S=80%, L=65%

// Wrong - using decimals will produce nearly black colors
TerminalColor.fromHSL(120, 0.8f, 0.65f);  // S=0.8%, L=0.65% ≈ black
```

### HSL Color Wheel

| Hue (degrees) | Color |
|---------------|-------|
| 0° | Red |
| 60° | Yellow |
| 120° | Green |
| 180° | Cyan |
| 240° | Blue |
| 300° | Magenta |
| 360° | Red (wraps around) |

### From HSL Values

**Remember:** Use percentages (0-100) for saturation and lightness, not decimals (0-1).

```java
// Foreground only - pure red at full saturation
TerminalColor red = TerminalColor.fromHSL(0, 100, 50);  // H=0°, S=100%, L=50%

// Both foreground and background
TerminalColor custom = TerminalColor.fromHSL(
    0, 100, 65,      // Foreground: light red (good for dark terminals)
    240, 80, 20      // Background: dark blue
);

// Optimal lightness for different backgrounds:
// - Dark terminals: use lightness 60-75 for visible colors
// - Light terminals: use lightness 25-40 for visible colors
int darkL = 65;  // 65% lightness for dark backgrounds
int lightL = 35; // 35% lightness for light backgrounds
```

### HSL Conversion Utilities

Convert between HSL and RGB:

```java
// HSL to RGB
int[] rgb = TerminalColor.hslToRgb(180, 80, 65);  // Cyan-ish
// Returns: [92, 230, 230]

// RGB to HSL
float[] hsl = TerminalColor.rgbToHsl(255, 128, 0);  // Orange
// Returns: [30.0, 100.0, 50.0] (H=30°, S=100%, L=50%)
```

### Getting HSL from True Color

```java
TerminalColor color = TerminalColor.fromRGB(255, 0, 0);

float[] fgHsl = color.getTextHSL();       // [0.0, 100.0, 50.0]
float[] bgHsl = color.getBackgroundHSL(); // null if not set
```

### Example: HSL Color Wheel Visualization

```java
// Display colors across the hue wheel
TerminalColorCapability cap = TerminalColorCapability.builder().build();
boolean isDark = cap.getTheme() == TerminalTheme.DARK;

int lightness = isDark ? 65 : 35;  // Adjust for theme
int saturation = 80;

for (int hue = 0; hue <= 360; hue += 30) {
    int[] rgb = TerminalColor.hslToRgb(hue, saturation, lightness);

    String sample = ANSIBuilder.builder(cap)
        .rgb(rgb[0], rgb[1], rgb[2])
        .append(String.format("Hue: %3d°", hue))
        .toString();

    System.out.println(sample);
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

// Get theme-appropriate colors for log levels
TerminalColor error = TerminalColor.forError(cap);          // Red
TerminalColor success = TerminalColor.forSuccess(cap);      // Green
TerminalColor warning = TerminalColor.forWarning(cap);      // Yellow
TerminalColor info = TerminalColor.forInfo(cap);            // Cyan/Blue
TerminalColor debug = TerminalColor.forDebug(cap);          // White/Gray (subdued)
TerminalColor trace = TerminalColor.forTrace(cap);          // Gray (least prominent)

// Other semantic colors
TerminalColor highlight = TerminalColor.forHighlight(cap);  // Emphasized
TerminalColor muted = TerminalColor.forMuted(cap);          // Dimmed
TerminalColor timestamp = TerminalColor.forTimestamp(cap);  // Cyan (for log timestamps)
TerminalColor message = TerminalColor.forMessage(cap);      // Magenta (for highlighted messages)
```

### How Theme Adaptation Works

| Method | Dark Theme | Light Theme | Use Case |
|--------|------------|-------------|----------|
| `forError()` | Bright red | Normal red | Error messages |
| `forSuccess()` | Bright green | Normal green | Success messages |
| `forWarning()` | Bright yellow | Normal yellow | Warnings |
| `forInfo()` | Bright cyan | Normal blue | Info messages |
| `forDebug()` | White | Gray | Debug messages (subdued) |
| `forTrace()` | 256-color gray | Gray | Trace messages (least prominent) |
| `forHighlight()` | Bright white | Black | Emphasized text |
| `forMuted()` | Normal white | Normal black | Secondary text |
| `forTimestamp()` | Bright cyan | Normal cyan | Log timestamps |
| `forMessage()` | Bright magenta | Normal magenta | Highlighted messages |

Log level colors follow a prominence hierarchy: **ERROR > WARN > INFO > DEBUG > TRACE**

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

## ANSI Color Utilities

The `ANSI` class provides utility methods for converting between RGB values and ANSI color codes. This is useful when you have RGB values (e.g., from OSC color queries) and need to find the closest ANSI equivalent.

### RGB to 256-Color Palette

Convert RGB to the nearest 256-color palette index:

```java
import org.aesh.terminal.utils.ANSI;

// Convert RGB to 256-color palette index (16-255)
int index = ANSI.rgbTo256Color(255, 128, 0);  // Returns palette index

// Use in escape sequence
connection.write("\u001B[38;5;" + index + "mColored text\u001B[0m");
```

### RGB to Basic ANSI Colors

Convert RGB to basic ANSI color codes (16 colors):

```java
// Get foreground color code (30-37 or 90-97)
// Brightness is automatically determined from luminance
int fgCode = ANSI.rgbToAnsiColor(255, 0, 0);  // Returns 91 (bright red)

// With explicit brightness control
int normalRed = ANSI.rgbToAnsiColor(255, 0, 0, false);  // Returns 31
int brightRed = ANSI.rgbToAnsiColor(255, 0, 0, true);   // Returns 91

// Get background color code (40-47 or 100-107)
int bgCode = ANSI.rgbToAnsiBackgroundColor(0, 0, 128);  // Returns blue background

// Get just the basic color code (30-37) without brightness offset
int basicCode = ANSI.rgbToBasicColorCode(0, 200, 0);  // Returns 32 (green)
```

### Basic Color Code Reference

| Code | Color | Bright Code |
|------|-------|-------------|
| 30 | Black | 90 |
| 31 | Red | 91 |
| 32 | Green | 92 |
| 33 | Yellow | 93 |
| 34 | Blue | 94 |
| 35 | Magenta | 95 |
| 36 | Cyan | 96 |
| 37 | White | 97 |

Background codes add 10 (40-47 normal, 100-107 bright).

### Brightness Detection

Check if an RGB color should use the bright variant:

```java
boolean isBright = ANSI.rgbIsBright(200, 200, 200);  // true
boolean isDark = ANSI.rgbIsBright(50, 50, 50);       // false
```

### Reverse Conversion: 256-Color to RGB

Convert a 256-color palette index back to RGB:

```java
// Get RGB values for a palette index
int[] rgb = ANSI.color256ToRgb(196);  // Returns [255, 0, 0] for bright red
int[] gray = ANSI.color256ToRgb(244); // Returns grayscale RGB

System.out.println("R=" + rgb[0] + " G=" + rgb[1] + " B=" + rgb[2]);
```

### Example: Using with OSC Color Queries

Combine color queries with ANSI conversion:

```java
// Query the terminal's background color
int[] bgColor = connection.queryBackgroundColor(500);

if (bgColor != null) {
    // Convert to 256-color index
    int paletteIndex = ANSI.rgbTo256Color(bgColor[0], bgColor[1], bgColor[2]);
    System.out.println("Background is palette color: " + paletteIndex);

    // Convert to basic ANSI code
    int ansiCode = ANSI.rgbToAnsiColor(bgColor[0], bgColor[1], bgColor[2]);
    System.out.println("Nearest basic ANSI code: " + ansiCode);

    // Check brightness for theme detection
    boolean isDark = !ANSI.rgbIsBright(bgColor[0], bgColor[1], bgColor[2]);
    System.out.println("Theme: " + (isDark ? "dark" : "light"));
}
```

### Example: Palette Color Round-Trip

```java
// Query a palette color from the terminal
int[] rgb = connection.queryPaletteColor(1, 500);

if (rgb != null) {
    // Convert back to palette index
    int index = ANSI.rgbTo256Color(rgb[0], rgb[1], rgb[2]);

    // Find nearest basic ANSI equivalent
    int ansiCode = ANSI.rgbToAnsiColor(rgb[0], rgb[1], rgb[2]);

    System.out.println("Palette 1: RGB(" + rgb[0] + "," + rgb[1] + "," + rgb[2] + ")");
    System.out.println("  -> 256-color index: " + index);
    System.out.println("  -> Basic ANSI code: " + ansiCode);
}
```

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

For more complex formatting, use `ANSIBuilder`. It provides a fluent API for building ANSI-formatted strings with support for basic colors, 256-color palette, true color RGB, and theme-aware semantic colors.

### Basic Usage

```java
import org.aesh.terminal.utils.ANSIBuilder;

String output = ANSIBuilder.builder()
    .redText("Error: ")
    .append("Something went wrong")
    .toString();

// Bold and colors
String styled = ANSIBuilder.builder()
    .bold("Title: ").append("Some text")
    .toString();
```

### Theme-Aware Builder

Create a builder with detected terminal capabilities for automatic theme adaptation:

```java
TerminalColorCapability cap = TerminalColorDetector.detect(connection);
ANSIBuilder builder = ANSIBuilder.builder(cap);

// Semantic colors adapt to the terminal theme
String output = builder
    .timestamp("2024-01-15 10:30:45").append(" ")
    .success("[INFO]").append(" ")
    .message("Application started")
    .toString();
```

### Semantic Color Methods

The builder provides theme-aware semantic colors for log levels:

```java
ANSIBuilder builder = ANSIBuilder.builder(cap);

// Log level colors (most to least prominent)
builder.error("Error message");      // Red (bright on dark, normal on light)
builder.warning("Warning message");  // Yellow
builder.success("Success message");  // Green
builder.info("Info message");        // Blue
builder.debug("Debug message");      // White/Gray (subdued)
builder.trace("Trace message");      // Gray (least prominent)

// Other semantic colors
builder.timestamp("10:30:45");       // Cyan (subdued, for log timestamps)
builder.message("Highlighted");      // Magenta (for highlighted messages)
```

### Customizing Semantic Colors

You can override the default semantic colors directly on the builder:

```java
// Override colors on the builder (takes precedence over capability)
ANSIBuilder builder = ANSIBuilder.builder()
    .errorCode(196)      // 256-color bright red
    .successCode(46)     // 256-color green
    .warningCode(208)    // 256-color orange
    .infoCode(75)        // 256-color cyan
    .debugCode(250)      // 256-color light gray
    .traceCode(240)      // 256-color dark gray
    .timestampCode(244)  // 256-color gray
    .messageCode(201);   // 256-color magenta

// Now semantic methods use your custom colors
builder.error("Custom error color");
builder.success("Custom success color");
```

Builder overrides take precedence over `TerminalColorCapability` settings:

```java
// Capability with default colors
TerminalColorCapability cap = TerminalColorDetector.detect(connection);

// Builder overrides specific colors
ANSIBuilder builder = ANSIBuilder.builder(cap)
    .errorCode(196)      // Override error to 256-color red
    .timestampCode(244); // Override timestamp to gray

// error() uses 196, other methods still use capability defaults
```

You can use basic ANSI codes (30-37, 90-97), 256-color palette indices (0-255), or RGB/hex values:

```java
// Basic ANSI codes
builder.errorCode(31);    // Basic red
builder.successCode(92);  // Bright green

// 256-color palette indices
builder.errorCode(196);   // 256-color bright red
builder.infoCode(75);     // 256-color cyan

// RGB values (true color)
builder.errorRgb(255, 0, 0);        // Red
builder.successRgb(0, 255, 0);      // Green
builder.warningRgb(255, 165, 0);    // Orange

// Hex values
builder.errorHex("#FF5733");        // Coral
builder.timestampHex("6495ED");     // Cornflower blue

// HSL values (hue, saturation%, lightness%)
builder.errorHsl(0, 80, 65);        // Light red for dark theme
builder.successHsl(120, 80, 65);    // Light green for dark theme
builder.warningHsl(60, 80, 65);     // Light yellow for dark theme
```

Color priority: **RGB/HSL override > code override > capability > default**.

### Extended Color Support

#### 256-Color Palette

```java
// Foreground color from 256-color palette
builder.color256(208).append("Orange text");

// Background color from 256-color palette
builder.bg256(17).append("Blue background");

// With text directly
builder.color256(196, "Red text");
```

#### True Color (24-bit RGB)

```java
// RGB foreground
builder.rgb(255, 87, 51).append("Orange text");

// RGB background
builder.bgRgb(30, 30, 30).append("Dark background");

// From hex string
builder.hex("#FF5733").append("Coral text");
builder.bgHex("#1E1E1E").append("Dark background");
```

#### HSL Colors

HSL is ideal for generating readable colors that adapt to terminal themes.

**Value ranges:** Hue is 0-360 degrees, Saturation and Lightness are 0-100 percentages (not 0-1 decimals).

```java
// HSL foreground (hue 0-360, saturation 0-100, lightness 0-100)
builder.hsl(0, 80, 65).append("Light red");    // Good for dark themes
builder.hsl(120, 80, 35).append("Dark green"); // Good for light themes

// HSL background
builder.bgHsl(240, 80, 20).append("Dark blue background");

// With text directly
builder.hsl(180, 80, 65, "Cyan text");

// Theme-adaptive pattern
boolean isDark = cap.getTheme() == TerminalTheme.DARK;
int lightness = isDark ? 65 : 35;
builder.hsl(0, 80, lightness).append("Theme-adaptive red");

// Override semantic colors with HSL
builder.timestampHsl(200, 30, 70)   // Light blue-gray for timestamps
       .messageHsl(280, 60, 65);    // Light purple for messages
```

**Common mistake:** Using decimal values like `0.8` instead of `80` will produce nearly black colors since `0.8%` lightness is almost black.

#### Bright Colors

```java
// Enable bright mode for the next color
builder.bright().redText().append("Bright red");

// Turn off bright mode
builder.brightOff();
```

### Convenience Methods

```java
// toLine() - returns string with newline appended
connection.write(builder.error("Error occurred").toLine());

// appendLine() - appends text with newline, returns builder
builder.appendLine("Line 1").appendLine("Line 2");

// reset() or clear() - clears builder for reuse
builder.reset();
builder.success("New content");
```

### Reusing the Builder

For efficiency, reuse a single builder instance:

```java
ANSIBuilder builder = ANSIBuilder.builder(cap);

// First message
connection.write(builder.error("Error occurred").toLine());

// Reset and reuse for next message
builder.reset();
connection.write(builder.success("Operation completed").toLine());

// Reset and reuse again
builder.reset();
connection.write(builder
    .timestamp("10:30:45").append(" ")
    .info("[INFO]").append(" ")
    .message("Processing...")
    .toLine());
```

### Complete Example

```java
TerminalColorCapability cap = TerminalColorDetector.detect(connection);
ANSIBuilder builder = ANSIBuilder.builder(cap);

// Log-style output with timestamps
connection.write(builder
    .timestamp("2024-01-15 10:30:45").append(" ")
    .success("[INFO]").append(" ")
    .message("Application started")
    .toLine());

builder.reset();
connection.write(builder
    .timestamp("2024-01-15 10:30:46").append(" ")
    .warning("[WARN]").append(" ")
    .append("Low memory condition")
    .toLine());

builder.reset();
connection.write(builder
    .timestamp("2024-01-15 10:30:47").append(" ")
    .error("[ERROR]").append(" ")
    .append("Connection failed")
    .toLine());

// Using 256-color for custom branding
builder.reset();
connection.write(builder
    .color256(208, "MyApp").append(": ")
    .append("Custom colored branding")
    .toLine());
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
| `fromHSL(h, s, l)` | Create with HSL foreground |
| `fromHSL(th, ts, tl, bh, bs, bl)` | Create with HSL foreground and background |
| `hslToRgb(h, s, l)` | Convert HSL to RGB array |
| `rgbToHsl(r, g, b)` | Convert RGB to HSL array |
| `forError(cap)` | Theme-aware red for errors |
| `forSuccess(cap)` | Theme-aware green for success |
| `forWarning(cap)` | Theme-aware yellow for warnings |
| `forInfo(cap)` | Theme-aware cyan/blue for info |
| `forHighlight(cap)` | Theme-aware emphasis color |
| `forMuted(cap)` | Theme-aware secondary text color |
| `forTimestamp(cap)` | Theme-aware cyan for log timestamps |
| `forMessage(cap)` | Theme-aware magenta for highlighted messages |

### Instance Methods

| Method | Description |
|--------|-------------|
| `isTrueColor()` | Check if using RGB mode |
| `getTextRGB()` | Get foreground RGB array or null |
| `getBackgroundRGB()` | Get background RGB array or null |
| `getTextHSL()` | Get foreground HSL array or null |
| `getBackgroundHSL()` | Get background HSL array or null |
| `forCapability(cap)` | Adapt to terminal's color depth |
| `toColor256()` | Convert RGB to 256-color palette |
| `toColor16()` | Convert RGB to basic 16 colors |
| `fullString()` | Get complete ANSI escape sequence |
| `toString()` | Get color codes (without ESC prefix) |

## See Also

- [Color Detection](color-detection) - Detecting terminal theme and color depth
- [Examples](examples) - Working code samples
