---
date: '2026-02-11T15:00:00+01:00'
draft: false
title: 'Progress Bar'
weight: 20
---

The progress bar utility displays progress feedback for long-running operations in the terminal. It supports multiple visual styles, optional labels, percentage and ratio display, and renders in-place using ANSI escape sequences.

## Quick Start

```java
import org.aesh.util.progress.ProgressBar;
import org.aesh.util.progress.ProgressBarStyle;

ProgressBar progress = ProgressBar.builder()
        .shell(invocation.getShell())
        .total(files.size())
        .label("Processing")
        .style(ProgressBarStyle.UNICODE)
        .build();

for (File file : files) {
    processFile(file);
    progress.step();
}
progress.complete();
```

Output (mid-progress):

```
Processing │████████░░░░░░░░│  52%
```

## Builder API

All configuration is done through the builder. Only `total` is required; everything else has sensible defaults.

```java
ProgressBar progress = ProgressBar.builder()
        .shell(invocation.getShell())  // Shell to write to
        .total(200)                    // total number of steps
        .label("Downloading")         // optional text before the bar
        .style(ProgressBarStyle.ASCII) // visual style (default: ASCII)
        .showPercentage(true)          // show XX% (default: true)
        .showRatio(true)               // show current/total (default: false)
        .width(80)                     // explicit width (default: auto from terminal)
        .build();
```

### Builder Options

| Method | Default | Description |
|--------|---------|-------------|
| `shell(Shell)` | `null` | The `Shell` instance for terminal output |
| `total(long)` | `0` | Total number of steps (supports large values via `long`) |
| `label(String)` | none | Text label displayed before the bar |
| `style(ProgressBarStyle)` | `ASCII` | Visual style for bar characters |
| `showPercentage(boolean)` | `true` | Whether to display the percentage |
| `showRatio(boolean)` | `false` | Whether to display `current/total` |
| `width(int)` | auto | Explicit terminal width; if not set, reads from `Shell.size()` or defaults to 80 |

## Update Methods

### `step()`

Increment progress by 1 and re-render the bar:

```java
for (File file : files) {
    processFile(file);
    progress.step();
}
```

### `step(long n)`

Increment progress by a given amount:

```java
progress.step(10);  // advance by 10 steps
```

### `update(long value)`

Set an absolute progress value:

```java
progress.update(75);  // jump to 75 out of total
```

### `complete()`

Mark the operation as finished. This fills the bar to 100% and prints a newline so subsequent output appears below:

```java
progress.complete();
```

### `complete(String message)`

Mark complete and replace the bar line with a custom message:

```java
progress.complete("Done! Processed 200 files.");
```

## Progress Bar Styles

Four predefined styles are available via `ProgressBarStyle`:

### ASCII

The default style using standard ASCII characters:

```java
ProgressBar.builder().style(ProgressBarStyle.ASCII)...
```

```
[####------] 40%
```

### UNICODE

Unicode block characters for a solid, modern look:

```java
ProgressBar.builder().style(ProgressBarStyle.UNICODE)...
```

```
│████░░░░░░│ 40%
```

### SIMPLE

Minimal style with equals signs and spaces:

```java
ProgressBar.builder().style(ProgressBarStyle.SIMPLE)...
```

```
[====      ] 40%
```

### ARROW

Like SIMPLE but with a `>` tip character at the progress front:

```java
ProgressBar.builder().style(ProgressBarStyle.ARROW)...
```

```
[===>      ] 40%
```

At 100% completion, the arrow tip disappears and the bar is fully filled with `=`.

### Style Characters

Each style defines five characters:

| Style | Fill | Empty | Left Bracket | Right Bracket | Tip |
|-------|------|-------|--------------|---------------|-----|
| `ASCII` | `#` | `-` | `[` | `]` | `#` |
| `UNICODE` | `█` | `░` | `│` | `│` | `█` |
| `SIMPLE` | `=` | ` ` | `[` | `]` | `=` |
| `ARROW` | `=` | ` ` | `[` | `]` | `>` |

## Display Options

### Label

An optional text label is rendered before the bar:

```java
ProgressBar.builder()
        .label("Downloading")
        .total(100)
        .build();
```

```
Downloading [####------]  40%
```

Without a label, the bar starts immediately:

```
[####------]  40%
```

### Percentage

Enabled by default. Disable with `showPercentage(false)`:

```java
ProgressBar.builder()
        .showPercentage(false)
        .total(100)
        .build();
```

```
[####------]
```

### Ratio

Disabled by default. Enable with `showRatio(true)` to show `current/total`:

```java
ProgressBar.builder()
        .showRatio(true)
        .total(200)
        .build();
```

```
[####------]  40% (80/200)
```

## Bar Width Calculation

The bar automatically sizes itself to fill the available terminal width. The layout is:

```
[label ] [left-bracket] [===bar===] [right-bracket] [ XX%] [ (current/total)]
```

The bar width is calculated as:

```
barWidth = terminalWidth - labelLength - brackets(2) - percentageLength - ratioLength
```

If the calculated bar width falls below 10 characters, it is clamped to a minimum of 10.

When no explicit width is set via `width()`, the builder reads the terminal width from `Shell.size().getWidth()`. If the shell is unavailable, it defaults to 80 columns.

## Using in Commands

Inside a command's `execute()` method, pass the `Shell` from the `CommandInvocation` to the builder:

```java
import org.aesh.command.Command;
import org.aesh.command.CommandDefinition;
import org.aesh.command.CommandResult;
import org.aesh.command.invocation.CommandInvocation;
import org.aesh.util.progress.ProgressBar;
import org.aesh.util.progress.ProgressBarStyle;

@CommandDefinition(name = "process", description = "Process files")
public class ProcessCommand implements Command<CommandInvocation> {

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        List<File> files = getFiles();

        ProgressBar progress = ProgressBar.builder()
                .shell(invocation.getShell())
                .total(files.size())
                .label("Processing")
                .style(ProgressBarStyle.UNICODE)
                .showRatio(true)
                .build();

        for (File file : files) {
            processFile(file);
            progress.step();
        }
        progress.complete("Done! Processed " + files.size() + " files.");

        return CommandResult.SUCCESS;
    }
}
```

The key calls in the chain:

| Call | Returns | Purpose |
|------|---------|---------|
| `invocation.getShell()` | `Shell` | Access the terminal |
| `shell.size()` | `Size` | Get terminal dimensions (used automatically) |
| `progress.step()` | `void` | Advance and re-render the bar |
| `progress.complete()` | `void` | Fill to 100% and move to next line |

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| `total` is 0 | Renders as 100% (avoids division by zero) |
| `current` exceeds `total` | Clamped to 100% |
| Very narrow terminal | Bar width clamped to minimum of 10 characters |
| `shell` is null | Update methods are no-ops; `render()` still works for testing |

## How It Works

Each call to `step()`, `update()`, or `complete()` re-renders the bar on the current line using ANSI escape sequences from `org.aesh.terminal.utils.ANSI`:

1. `ANSI.CURSOR_START` — moves the cursor to the beginning of the line
2. `ANSI.ERASE_WHOLE_LINE` — clears the entire line
3. `Shell.write(String)` — writes the formatted bar string

This creates a smooth in-place update effect without scrolling the terminal.

## API Reference

### ProgressBar

| Method | Description |
|--------|-------------|
| `ProgressBar.builder()` | Create a builder for fluent configuration |
| `update(long)` | Set absolute progress and re-render |
| `step()` | Increment by 1 and re-render |
| `step(long)` | Increment by n and re-render |
| `complete()` | Fill to 100% and print a newline |
| `complete(String)` | Replace the bar with a completion message |

### ProgressBarStyle

| Style | Description |
|-------|-------------|
| `ASCII` | `#` fill, `-` empty, `[` `]` brackets |
| `UNICODE` | `█` fill, `░` empty, `│` `│` brackets |
| `SIMPLE` | `=` fill, space empty, `[` `]` brackets |
| `ARROW` | `=` fill with `>` tip, space empty, `[` `]` brackets |

## See Also

- [Table Display](table) — structured data rendering utility
- [CommandInvocation API](command-invocation) — accessing the Shell in commands
