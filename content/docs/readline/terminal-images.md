---
date: '2026-02-03T12:00:00+01:00'
draft: false
title: 'Terminal Images'
weight: 12
---

Æsh Readline provides support for displaying images directly in the terminal using multiple graphics protocols. This allows CLI applications to show logos, charts, screenshots, and other visual content.

## Overview

The terminal image API supports three major graphics protocols:

| Protocol | Terminals |
|----------|-----------|
| **Kitty Graphics** | Kitty, Ghostty, Konsole (partial) |
| **iTerm2 Inline Images** | iTerm2, WezTerm, Mintty, VSCode Terminal, Tabby, Hyper |
| **Sixel Graphics** | xterm, mlterm, foot, Windows Terminal, Contour |

The API automatically detects which protocol the terminal supports and generates the appropriate escape sequences.

## Quick Start

```java
import org.aesh.terminal.image.TerminalImage;
import org.aesh.terminal.image.TerminalImageBuilder;
import org.aesh.terminal.tty.TerminalConnection;

import java.nio.file.Files;
import java.nio.file.Path;

TerminalConnection conn = new TerminalConnection();

// Load image from file
byte[] imageData = Files.readAllBytes(Path.of("logo.png"));

// Build image with auto-detected protocol
TerminalImage image = TerminalImageBuilder.builder(conn.device())
    .data(imageData)
    .filename("logo.png")
    .widthCells(40)  // Display width in terminal cells
    .build();

// Display the image
if (image != null) {
    conn.write(image.encode());
}

conn.close();
```

## Checking Image Support

Before displaying images, check if the terminal supports graphics:

```java
import org.aesh.terminal.image.ImageProtocol;

ImageProtocol protocol = conn.device().getImageProtocol();

switch (protocol) {
    case KITTY:
        System.out.println("Using Kitty graphics protocol");
        break;
    case ITERM2:
        System.out.println("Using iTerm2 inline images");
        break;
    case SIXEL:
        System.out.println("Using Sixel graphics");
        break;
    case NONE:
        System.out.println("No image support detected");
        break;
}

// Simple check
if (conn.device().supportsImages()) {
    // Display image
}
```

### Protocol Detection from Environment

The `ImageProtocolDetector` uses [`TerminalEnvironment`](terminal-environment) for detection:

```java
import org.aesh.terminal.image.ImageProtocolDetector;
import org.aesh.terminal.Device;

// Detect from current environment (uses TerminalEnvironment)
ImageProtocol protocol = ImageProtocolDetector.detectFromEnvironment();

// Get protocol for a specific terminal type
Device.TerminalType type = Device.TerminalType.KITTY;
ImageProtocol kittyProtocol = ImageProtocolDetector.getProtocolForTerminalType(type);
// Returns: ImageProtocol.KITTY
```

### Protocol by Terminal Type

| Terminal Type | Image Protocol |
|---------------|----------------|
| `KITTY`, `GHOSTTY`, `KONSOLE` | KITTY |
| `ITERM2`, `WEZTERM`, `MINTTY`, `VSCODE`, `TABBY`, `HYPER` | ITERM2 |
| `FOOT`, `CONTOUR` | SIXEL |
| Others | NONE |

## TerminalImageBuilder

The `TerminalImageBuilder` class provides a fluent API for creating images:

### Creating a Builder

```java
// Auto-detect protocol from terminal device
TerminalImageBuilder builder = TerminalImageBuilder.builder(conn.device());

// Or specify a protocol explicitly
TerminalImageBuilder builder = TerminalImageBuilder.builder(ImageProtocol.ITERM2);
```

### Setting Image Data

```java
// From file path
builder.file(Path.of("image.png"));

// From byte array
builder.data(imageBytes);

// Set filename (used by iTerm2 protocol)
builder.filename("image.png");
```

### Setting Dimensions

```java
// Width/height in terminal cells
builder.widthCells(40);
builder.heightCells(20);

// Width/height in pixels
builder.widthPixels(800);
builder.heightPixels(600);

// Preserve aspect ratio (iTerm2 only)
builder.preserveAspectRatio(true);
```

### Building and Displaying

```java
TerminalImage image = builder.build();

if (image != null) {
    String escapeSequence = image.encode();
    conn.write(escapeSequence);
}
```

## Protocol-Specific Features

### iTerm2 Protocol

The iTerm2 protocol is widely supported and offers several options:

```java
import org.aesh.terminal.image.ITermImage;

ITermImage image = new ITermImage(imageData)
    .filename("chart.png")
    .widthCells(60)
    .heightCells(30)
    .preserveAspectRatio(true);

conn.write(image.encode());
```

**Sizing options:**
- `widthCells(int)` / `heightCells(int)` - Size in terminal cells
- `widthPixels(int)` / `heightPixels(int)` - Size in pixels
- `widthPercent(int)` / `heightPercent(int)` - Size as percentage of terminal

**Other options:**
- `preserveAspectRatio(boolean)` - Maintain image proportions
- `useStTerminator(boolean)` - Use ST instead of BEL terminator

### Kitty Graphics Protocol

The Kitty protocol provides high-quality graphics with advanced features:

```java
import org.aesh.terminal.image.KittyImage;

KittyImage image = new KittyImage(imageData)
    .widthCells(50)
    .heightCells(25)
    .zIndex(1)  // Layering support
    .suppressResponse(true);

conn.write(image.encode());
```

**Features:**
- Automatic PNG conversion (Kitty requires PNG format)
- Chunked transmission for large images
- Z-index layering support
- Response suppression to avoid input clutter

**Options:**
- `widthCells(int)` / `heightCells(int)` - Display dimensions
- `sourceDimensions(int width, int height)` - For raw RGB/RGBA data
- `zIndex(int)` - Layer ordering
- `suppressResponse(boolean)` - Suppress terminal responses (default: true)

### Sixel Graphics

Sixel is a legacy but widely supported protocol:

```java
import org.aesh.terminal.image.SixelImage;

SixelImage image = new SixelImage(imageData)
    .maxWidth(800)
    .maxHeight(600)
    .maxColors(256)  // Color palette size (2-256)
    .useRle(true);   // Run-length encoding compression

conn.write(image.encode());
```

**Features:**
- Automatic image scaling with aspect ratio preservation
- Color quantization (reduces colors to fit palette)
- RLE compression for smaller output
- Works in many legacy terminals

**Options:**
- `maxWidth(int)` / `maxHeight(int)` - Maximum dimensions in pixels
- `maxColors(int)` - Palette size (2-256, affects quality vs size)
- `useRle(boolean)` - Enable run-length encoding compression

## Image Format Support

The API supports multiple image formats:

| Format | Kitty | iTerm2 | Sixel |
|--------|-------|--------|-------|
| PNG | Yes (native) | Yes | Yes |
| JPEG | Converted to PNG | Yes | Yes |
| GIF | Converted to PNG | Yes | Yes |

For Kitty protocol, non-PNG images are automatically converted to PNG.

### Format Detection

```java
import org.aesh.terminal.image.ImageUtils;

byte[] data = Files.readAllBytes(Path.of("image.jpg"));

if (ImageUtils.isPng(data)) {
    System.out.println("PNG format");
} else if (ImageUtils.isJpeg(data)) {
    System.out.println("JPEG format");
} else if (ImageUtils.isGif(data)) {
    System.out.println("GIF format");
}

// Convert any format to PNG
byte[] pngData = ImageUtils.toPng(data);
```

## Complete Example

Here's a complete example that displays an image with fallback handling:

```java
import org.aesh.terminal.image.*;
import org.aesh.terminal.tty.TerminalConnection;
import java.nio.file.Files;
import java.nio.file.Path;

public class ImageDemo {
    public static void main(String[] args) throws Exception {
        TerminalConnection conn = new TerminalConnection();

        // Check for image support
        ImageProtocol protocol = conn.device().getImageProtocol();

        if (protocol == ImageProtocol.NONE) {
            conn.write("This terminal does not support images.\n");
            conn.write("Try running in: Kitty, iTerm2, WezTerm, or VSCode\n");
            conn.close();
            return;
        }

        conn.write("Detected protocol: " + protocol + "\n\n");

        // Load and display image
        Path imagePath = Path.of(args[0]);
        byte[] imageData = Files.readAllBytes(imagePath);

        TerminalImage image = TerminalImageBuilder.builder(conn.device())
            .data(imageData)
            .filename(imagePath.getFileName().toString())
            .widthCells(60)
            .preserveAspectRatio(true)
            .build();

        if (image != null) {
            conn.write(image.encode());
            conn.write("\n\nImage displayed successfully!\n");
        } else {
            conn.write("Failed to create image.\n");
        }

        conn.close();
    }
}
```

## Protocol Detection

The API uses multiple methods to detect the graphics protocol, from most to least accurate:

### 1. DA1 Device Attributes (Most Accurate for Sixel)

Query the terminal's device attributes for authoritative Sixel detection:

```java
// Use DA1 query for accurate Sixel detection
ImageProtocol protocol = connection.queryImageProtocol(500);

switch (protocol) {
    case KITTY:
        // Kitty graphics protocol
        break;
    case ITERM2:
        // iTerm2 inline images
        break;
    case SIXEL:
        // Authoritatively detected via DA1 parameter 4
        break;
    case NONE:
        // No image support
        break;
}
```

The DA1 response includes parameter 4 when Sixel is supported, providing reliable detection even for terminals that don't set specific environment variables.

### 2. Environment Variables

| Variable | Protocol |
|----------|----------|
| `KITTY_WINDOW_ID` | Kitty |
| `GHOSTTY_RESOURCES_DIR` | Kitty |
| `ITERM_SESSION_ID` | iTerm2 |
| `WEZTERM_PANE` | iTerm2 |
| `TERM_PROGRAM=vscode` | iTerm2 |

### 3. TERM Variable Patterns

| TERM contains | Protocol |
|---------------|----------|
| `kitty` | Kitty |
| `ghostty` | Kitty |
| `konsole` | Kitty |
| `iterm`, `wezterm`, `mintty` | iTerm2 |
| `vscode`, `tabby`, `hyper` | iTerm2 |
| `mlterm`, `foot`, `contour` | Sixel |

### Heuristic vs Query-Based Detection

```java
// Heuristic detection (fast, no terminal query)
// Uses environment variables and TERM type
Device device = connection.device();
ImageProtocol protocol = device.getImageProtocol();

// Query-based detection (accurate, queries terminal)
// Uses DA1 for authoritative Sixel detection
ImageProtocol protocol = connection.queryImageProtocol(500);
```

Use heuristic detection when you need immediate results. Use query-based detection when accuracy is important and you can wait for the terminal response.

### Manual Protocol Override

If automatic detection doesn't work, specify the protocol explicitly:

```java
TerminalImage image = TerminalImageBuilder.builder(ImageProtocol.ITERM2)
    .data(imageData)
    .widthCells(40)
    .build();
```

## Supported Terminals

| Terminal | Protocol | Notes |
|----------|----------|-------|
| Kitty | Kitty | Full support |
| Ghostty | Kitty | Full support |
| iTerm2 | iTerm2 | Full support |
| WezTerm | iTerm2 | Full support |
| VSCode Terminal | iTerm2 | Full support |
| Mintty | iTerm2 | Full support |
| Tabby | iTerm2 | Full support |
| Hyper | iTerm2 | Full support |
| Konsole | Kitty | Partial support |
| xterm | Sixel | Requires `-ti vt340` flag |
| mlterm | Sixel | Full support |
| foot | Sixel | Full support |
| Windows Terminal | Sixel | Full support |
| Contour | Sixel | Full support |
| Alacritty | None | No image support |
| Apple Terminal | None | No image support |
| GNOME Terminal | None | No image support |

## Performance Considerations

1. **Image Size**: Large images generate large escape sequences. Consider resizing before display.

2. **Sixel Colors**: Reducing `maxColors` improves performance but reduces quality.

3. **PNG Conversion**: Kitty protocol requires PNG, so JPEG/GIF images are converted at runtime.

4. **Chunked Transmission**: Kitty protocol sends large images in 4KB chunks to avoid buffer issues.

## Troubleshooting

### Image Not Displayed

1. **Check terminal support**: Run the example to see detected protocol
2. **Try explicit protocol**: Use `TerminalImageBuilder.builder(ImageProtocol.ITERM2)`
3. **Check image format**: Ensure the image file is valid

### Garbled Output

1. **Terminal doesn't support the protocol**: Try a different terminal
2. **Image too large**: Reduce dimensions with `widthCells()` or `maxWidth()`

### Colors Look Wrong (Sixel)

1. **Increase color palette**: Use `.maxColors(256)` for better quality
2. **Try different terminal**: Some terminals have better Sixel support
