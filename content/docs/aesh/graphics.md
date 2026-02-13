---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Terminal Graphics'
weight: 22
---

The Graphics API provides primitives for drawing shapes, lines, and text at arbitrary positions in the terminal. It uses ANSI cursor addressing to place content at specific coordinates, with support for colors and the alternate screen buffer.

## Graphics Interface

```java
public interface Graphics {
    void flush();
    void clear();
    void clearAndShowCursor();

    TerminalColor getColor();
    void setColor(TerminalColor color);

    TerminalTextStyle getTextStyle();
    void setTextStyle(TerminalTextStyle textStyle);

    void drawRect(int x, int y, int width, int height);
    void fillRect(int x, int y, int width, int height);
    void drawLine(int x1, int y1, int x2, int y2);
    void drawCircle(int x, int y, int radius);
    void drawString(String str, int x, int y);
}
```

| Method | Description |
|--------|-------------|
| `drawRect(x, y, w, h)` | Draw a rectangle outline at (x, y) with the given width and height |
| `fillRect(x, y, w, h)` | Fill a rectangle with the current background color |
| `drawLine(x1, y1, x2, y2)` | Draw a line between two points |
| `drawCircle(x, y, radius)` | Draw a circle centered at (x, y) |
| `drawString(str, x, y)` | Draw text at position (x, y) |
| `setColor(color)` | Set foreground and background colors for subsequent drawing |
| `clear()` | Clear the entire screen |
| `clearAndShowCursor()` | Clear the screen and restore the cursor |
| `flush()` | Flush pending output to the terminal |

Coordinates are column (x) and row (y), with (0, 0) at the top-left corner of the terminal.

## Setup

Create a `GraphicsConfiguration` from a terminal `Connection`, then obtain a `Graphics` instance:

```java
import org.aesh.graphics.AeshGraphicsConfiguration;
import org.aesh.graphics.Graphics;
import org.aesh.graphics.GraphicsConfiguration;

GraphicsConfiguration gc = new AeshGraphicsConfiguration(connection);
Graphics g = gc.getGraphics();
```

The `GraphicsConfiguration` also provides `getBounds()` to query the terminal size, which is useful for positioning elements relative to the terminal dimensions.

## Example Command

```java
@CommandDefinition(name = "draw", description = "Draw shapes")
public class DrawCommand implements Command<CommandInvocation> {

    private final GraphicsConfiguration gc;

    public DrawCommand(Connection connection) {
        this.gc = new AeshGraphicsConfiguration(connection);
    }

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Switch to alternate buffer to preserve scroll history
        invocation.getShell().enableAlternateBuffer();

        Graphics g = gc.getGraphics();

        // Blue rectangle
        g.setColor(new TerminalColor(Color.BLUE, Color.DEFAULT));
        g.drawRect(5, 2, 30, 8);
        g.flush();

        // Red filled area
        g.setColor(new TerminalColor(Color.DEFAULT, Color.RED));
        g.fillRect(40, 3, 15, 6);
        g.flush();

        // Cyan text
        g.setColor(new TerminalColor(Color.CYAN, Color.DEFAULT));
        g.drawString("Hello from Aesh Graphics!", 10, 12);
        g.flush();

        // Green circle
        g.setColor(new TerminalColor(Color.GREEN, Color.DEFAULT));
        g.drawCircle(60, 8, 5);
        g.flush();

        // Wait for 'q' to exit
        try {
            while (!invocation.input().equals(Key.q)) {
                // wait
            }
        } catch (InterruptedException ignored) {
        }

        // Restore terminal
        g.clearAndShowCursor();
        invocation.getShell().enableMainBuffer();
        return CommandResult.SUCCESS;
    }
}
```

Register the command with the terminal connection:

```java
TerminalConnection connection = new TerminalConnection(
    Charset.defaultCharset(), System.in, System.out, null);

CommandRegistry registry = AeshCommandRegistryBuilder.builder()
    .command(new DrawCommand(connection))
    .create();
```

## Colors

Use `TerminalColor` to set foreground and background colors:

```java
import org.aesh.terminal.formatting.Color;
import org.aesh.terminal.formatting.TerminalColor;

// Foreground only
g.setColor(new TerminalColor(Color.RED, Color.DEFAULT));

// Both foreground and background
g.setColor(new TerminalColor(Color.WHITE, Color.BLUE));
```

Available colors: `BLACK`, `RED`, `GREEN`, `YELLOW`, `BLUE`, `MAGENTA`, `CYAN`, `WHITE`, `DEFAULT`.

Color changes apply to all subsequent drawing operations until `setColor()` is called again.

## Alternate Screen Buffer

For full-screen graphics, use the alternate screen buffer. This preserves the user's terminal history and restores it when your command finishes:

```java
// Enter alternate buffer
invocation.getShell().enableAlternateBuffer();

// ... draw graphics ...

// Return to main buffer (restores previous content)
invocation.getShell().enableMainBuffer();
```

## Animation

Combine `clear()`, drawing, `flush()`, and `Thread.sleep()` for simple animations:

```java
g.setColor(new TerminalColor(Color.DEFAULT, Color.GREEN));
for (int x = 0; x < 80; x++) {
    g.clear();
    g.fillRect(x, 10, 10, 5);
    g.flush();
    Thread.sleep(50);
}
```

## Bounds Checking

The graphics implementation clips drawing to the terminal bounds. Use `GraphicsConfiguration.getBounds()` to adapt your layout to the terminal size:

```java
Size bounds = gc.getBounds();
int centerX = bounds.getWidth() / 2;
int centerY = bounds.getHeight() / 2;
g.drawString("Centered", centerX - 4, centerY);
```
