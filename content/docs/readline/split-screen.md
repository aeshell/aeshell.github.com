---
date: '2026-06-10T15:00:00+01:00'
draft: false
title: 'Split Screen'
weight: 27
---

{{< callout type="warning" >}}
**Experimental API** — this feature is under active development and the API may change.
{{< /callout >}}

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

## printAbove Integration

When split screen is active, `connection.printAbove()` automatically routes to the top region:

```java
// During split screen, this writes to the top region
connection.printAbove("[INFO] Build completed");

// Direct region write also works
split.topRegion().writeln("[DEBUG] Processing...");
```

This means existing code using `printAbove()` for async output will automatically use the top region when split screen is enabled.

## Resize Handling

The split screen automatically adjusts when the terminal is resized:
- The split ratio is maintained
- The separator is redrawn at the new position
- Both regions are resized proportionally
- If the terminal becomes too small (below 7 rows), the split is automatically suspended and restored when the terminal grows back

## Suspend and Resume

When a command needs the full screen (editor, pager), suspend the split:

```java
split.suspend();   // clears split, resets scroll region
// ... full-screen command runs ...
split.resume();    // restores split layout, redraws
```

The split also auto-suspends when the terminal is resized too small, and auto-resumes when space is available again.

## Output Routing

`Connection.setCurrentRegion()` can route `connection.write()` to a specific region:

```java
connection.setCurrentRegion(split.topRegion());
connection.write("goes to top");

connection.setCurrentRegion(null);  // reset to default
connection.write("goes to terminal directly");
```

## Terminal Cleanup

The split screen registers a JVM shutdown hook to reset the terminal state (DECSTBM scroll region) on abnormal exit, preventing a corrupted terminal.

## Tab Completion and History Search

Tab completion and fuzzy history search (Ctrl+R) work normally within the bottom region. They are constrained to the bottom region's bounds automatically.

## Example

See `SplitScreenExample` in the examples directory. Commands:
- `log <msg>` — write to top region
- `above <msg>` — test printAbove routing to top region
- `clear` — clear top region
- `help` — show commands
- Tab completion and Ctrl+R work in the bottom region
