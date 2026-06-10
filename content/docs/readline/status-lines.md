---
date: '2026-06-10T15:00:00+01:00'
draft: false
title: 'Status Lines'
weight: 26
---

Persistent status lines displayed between scrolling output and the readline prompt. Status lines stay visible across `printAbove()` calls and are redrawn automatically.

## Usage

```java
// Register status lines with different priorities
// Lower priority = top, higher priority = closer to prompt
StatusLine buildStatus = connection.registerStatusLine(100);
StatusLine testStatus = connection.registerStatusLine(200);

// Set initial messages
buildStatus.setMessage("\033[33m[Build]\033[0m Compiling...");
testStatus.setMessage("\033[36m[Tests]\033[0m 42 passed, 0 failed");

// Update from any thread
buildStatus.setMessage("\033[32m[Build]\033[0m Complete!");

// Remove when no longer needed
testStatus.close();
```

## Display Order

Status lines are rendered in priority order between the scrolling output area and the readline prompt:

```
[scrolling output from printAbove()]
[Build] Compiling...              ← priority 100
[Tests] 42 passed, 0 failed      ← priority 200
$ user types here_                ← readline prompt
```

## Thread Safety

`StatusLine.setMessage()` and `StatusLine.close()` are thread-safe. Status updates trigger a redraw of the status area and prompt.

## ANSI Support

Status line messages can contain ANSI escape sequences:

```java
statusLine.setMessage("\033[1;32m●\033[0m Service running");
```

## Hiding a Status Line

Setting the message to null or empty hides the line without closing it:

```java
statusLine.setMessage(null);    // hidden, can be shown again later
statusLine.setMessage("back");  // visible again
```

## Example

See `StatusLineExample` in the examples directory for a complete working example simulating a build tool with status lines and log output.
