---
date: '2026-06-10T15:00:00+01:00'
draft: false
title: 'Split Screen'
weight: 27
---

{{% alert context="warning" %}}
**Experimental API** — this feature is under active development and the API may change.
{{% /alert %}}

Split the terminal into independently scrolling regions: a top region for output (logs, monitoring) and a bottom region for readline input.

## Usage

```java
// Split: top 2/3 for output, bottom 1/3 for readline
SplitScreen split = connection.splitScreen(0.67);
ScreenRegion logRegion = split.topRegion();

// Write to the top region from any thread
logRegion.writeln("[INFO] Server started on port 8080");
logRegion.writeln("[DEBUG] Processing request...");

// Readline operates normally in the bottom region
readline.readline(connection, "$ ", line -> { ... });

// Close when done — restores full screen
split.close();
```

## Architecture

The split screen uses a hybrid approach:
- **Top region**: rendered via cursor addressing (`ESC [row;colH`), with its own scrollback buffer
- **Bottom region**: constrained by DECSTBM scroll region so readline doesn't scroll past the separator
- **Separator**: tmux-style line between regions

```
┌─────────────────────────────────────────┐
│ [INFO] Server started                   │  ← Top region
│ [DEBUG] Processing request...           │    (scrolls independently)
│ [INFO] Request completed in 23ms        │
├─────────────────────────────────────────┤  ← Separator
│ $ user types here_                      │  ← Bottom region
│                                         │    (readline)
└─────────────────────────────────────────┘
```

## Region Sizes

Each region has its own size. `connection.size()` returns the bottom region's size when split is active, so readline wraps correctly:

```java
ScreenRegion top = split.topRegion();
ScreenRegion bottom = split.bottomRegion();
System.out.println(top.size());    // e.g., 80x16
System.out.println(bottom.size()); // e.g., 80x7
```

## Scrollback Buffer

Each region maintains a scrollback buffer (default 1000 lines). When the visible area fills up, older lines scroll off the top but are retained in the buffer for redrawing on resize.

## Separator

The separator line between regions can be customized:

```java
split.setSeparator("═");    // double line
split.setSeparator("- ");   // dashed
split.setSeparator(null);   // no separator
```

## Suspend and Resume

When a command needs the full screen (editor, pager), suspend the split:

```java
split.suspend();   // clears split, resets scroll region
// ... full-screen command runs ...
split.resume();    // restores split layout, redraws
```

## Output Routing

`Connection.setCurrentRegion()` can route `connection.write()` to a specific region:

```java
connection.setCurrentRegion(split.topRegion());
connection.write("goes to top");

connection.setCurrentRegion(null);  // reset to default
connection.write("goes to terminal directly");
```

## Example

See `SplitScreenExample` in the examples directory.
