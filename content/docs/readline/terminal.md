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
