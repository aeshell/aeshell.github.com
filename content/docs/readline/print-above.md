---
date: '2026-06-10T15:00:00+01:00'
draft: false
title: 'Print Above'
weight: 25
---

Print text above the current prompt without disrupting the user's input. Essential for displaying asynchronous output (log messages, notifications, background task results) while the user is typing.

## Usage

```java
// From any thread, while readline is active
connection.printAbove("Build completed successfully");
connection.printAbove("\033[33m[WARN]\033[0m Disk space low");
```

When a readline session is active, `printAbove()`:
1. Erases the current prompt and buffer from the screen
2. Prints the text at the current line
3. Redraws the prompt and buffer with the cursor in the correct position

When no readline session is active, the text is simply written to the terminal with a newline.

## Thread Safety

`printAbove()` is thread-safe and can be called from any thread. It synchronizes with readline's input processing to prevent interleaving.

```java
// Background thread sending notifications
ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
scheduler.scheduleAtFixedRate(() -> {
    connection.printAbove("[" + LocalTime.now() + "] Heartbeat");
}, 5, 5, TimeUnit.SECONDS);
```

## ANSI Support

The text can contain ANSI escape sequences for colors and styling:

```java
connection.printAbove("\033[32m[OK]\033[0m All tests passed");
connection.printAbove("\033[1;31m[ERROR]\033[0m Connection refused");
```

## With Synchronized Output

When Mode 2026 (synchronized output) is supported, `printAbove()` wraps the erase/redraw in BSU/ESU to prevent flicker.

## Example

See `PrintAboveExample` in the examples directory for a complete working example with background notifications.
