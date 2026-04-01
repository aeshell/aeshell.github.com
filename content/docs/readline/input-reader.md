---
date: '2026-03-20T10:00:00+01:00'
draft: false
title: 'InputReader'
weight: 10
---

`InputReader` is a utility that bridges the terminal's push-style input (`Consumer<int[]>`) to Java's pull-style `java.io.Reader`. This is useful when you need to read terminal input using standard Java I/O APIs — for example, wrapping a `BufferedReader` around terminal input, or integrating with libraries that expect a `Reader`.

## Quick Start

The simplest way to create an `InputReader` is with the `asReader()` factory method:

```java
import org.aesh.terminal.utils.InputReader;
import org.aesh.terminal.tty.TerminalConnection;

TerminalConnection connection = new TerminalConnection();
connection.enterRawMode();
connection.openNonBlocking();

InputReader reader = InputReader.asReader(connection);

// Now use standard Reader methods
char[] buf = new char[256];
int n = reader.read(buf, 0, buf.length);
String input = new String(buf, 0, n);
```

This creates an `InputReader` and wires it to the connection's stdin handler in one step.

## How It Works

`Connection` delivers input as code point arrays to a `Consumer<int[]>` handler. `InputReader` converts these into `char`s (splitting supplementary code points into surrogate pairs) and buffers them in a bounded queue. You then read from the queue using standard `Reader` methods or the timeout-aware methods.

```
Connection stdin ──► push(int) ──► LinkedBlockingQueue<Character> ──► read()
                     (splits supplementary                           (blocking/
                      into surrogates)                                timeout)
```

## Creating an InputReader

### From a Connection (recommended)

```java
// Default queue capacity (4096)
InputReader reader = InputReader.asReader(connection);

// Custom queue capacity
InputReader reader = InputReader.asReader(connection, 8192);
```

### Manual Wiring

If you need more control over how input is fed to the reader:

```java
InputReader reader = new InputReader();

// Wire it yourself
connection.setStdinHandler(cps -> {
    for (int cp : cps) {
        reader.push(cp);
    }
});
```

### Standalone (no Connection)

You can also push data manually — useful for testing or non-terminal input sources:

```java
InputReader reader = new InputReader();
reader.push("Hello");
reader.push(0x1F600);  // emoji, split into surrogate pair
reader.push('!');
```

## Reading Input

### Standard Reader Methods

`InputReader` extends `java.io.Reader`, so all standard methods work:

```java
// Read into a char array — blocks on first char, drains rest without blocking
char[] buf = new char[1024];
int n = reader.read(buf, 0, buf.length);

// Check if input is available without blocking
if (reader.ready()) {
    // Data available
}
```

### Read with Timeout

Read a single character with a millisecond timeout. Returns `InputReader.TIMEOUT` (-2) if no input arrives, or `InputReader.EOF` (-1) on end of stream:

```java
int ch = reader.read(50);  // 50ms timeout
if (ch == InputReader.TIMEOUT) {
    // No input within timeout
} else if (ch == InputReader.EOF) {
    // End of stream
} else {
    System.out.println("Got: " + (char) ch);
}
```

This is particularly useful for game loops or interactive applications that need to check for input without blocking indefinitely:

```java
while (running) {
    int ch = reader.read(16);  // ~60fps
    if (ch != InputReader.TIMEOUT && ch != InputReader.EOF) {
        handleInput((char) ch);
    }
    updateGame();
    render();
}
```

### Read Code Points

Read a full Unicode code point, automatically reassembling surrogate pairs:

```java
OptionalInt cp = reader.readCodePoint(1, TimeUnit.SECONDS);
if (cp.isPresent()) {
    System.out.println("Code point: U+" + Integer.toHexString(cp.getAsInt()));
}
```

## Push Methods

The `push` methods are the producer side — they feed data into the reader's internal queue:

| Method | Description |
|--------|-------------|
| `push(char ch)` | Push a single char |
| `push(int codePoint)` | Push a Unicode code point (supplementary code points split into surrogate pairs) |
| `push(CharSequence csq)` | Push a string or other CharSequence |

Push is non-blocking. If the queue is full, excess characters are silently dropped. After `close()`, pushes are silently ignored.

## Bounded Queue

The internal queue has a bounded capacity (default 4096 chars) to prevent unbounded memory growth if the reader falls behind. You can configure the capacity via the constructor:

```java
// Small buffer for memory-constrained environments
InputReader reader = new InputReader(256);

// Large buffer for high-throughput scenarios
InputReader reader = new InputReader(16384);
```

## Closing

```java
reader.close();
```

Closing the reader:
- Unblocks any thread waiting on `read()` (returns -1 / EOF)
- Clears the queue
- Causes subsequent `read()`, `ready()`, and `readCodePoint()` to throw `IOException`
- Causes subsequent `push()` calls to be silently ignored

## Complete Example

A simple terminal echo application using `InputReader`:

```java
import org.aesh.terminal.tty.Signal;
import org.aesh.terminal.tty.TerminalConnection;
import org.aesh.terminal.utils.InputReader;

public class EchoApp {
    public static void main(String[] args) throws Exception {
        try (TerminalConnection connection = new TerminalConnection()) {
            connection.enterRawMode();
            connection.openNonBlocking();

            InputReader reader = InputReader.asReader(connection);

            connection.setSignalHandler(signal -> {
                if (signal == Signal.INT) {
                    reader.close();
                }
            });

            int ch;
            while ((ch = reader.read(100)) != InputReader.EOF) {
                if (ch != InputReader.TIMEOUT) {
                    connection.write(String.valueOf((char) ch));
                    if (ch == '\r') {
                        connection.write("\n");
                    }
                }
            }
        }
    }
}
```
