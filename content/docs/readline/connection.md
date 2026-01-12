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
