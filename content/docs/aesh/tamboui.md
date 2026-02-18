---
date: '2026-02-18T15:00:00+01:00'
draft: false
title: 'TamboUI Integration'
weight: 23
---

The `aesh-tamboui` module integrates [TamboUI](https://github.com/tamboui-dev/tamboui), a Java TUI framework inspired by [ratatui](https://ratatui.rs/), with aesh commands. It lets you build rich terminal user interfaces — dashboards, charts, tables, and interactive views — that run inside aesh commands and return cleanly to the aesh prompt when done.

## Overview

TamboUI provides a widget-based rendering model with layouts, styles, and an event loop. The `aesh-tamboui` module bridges the two frameworks by sharing aesh's terminal `Connection` with TamboUI's `AeshBackend`. This means your TUI commands get full access to TamboUI's widget library while aesh continues to manage the terminal lifecycle.

Three integration levels are available:

| Level | Class | Style | Best for |
|-------|-------|-------|----------|
| Utility | `TuiSupport` | Manual setup | Advanced users who want full control |
| Event loop | `TuiCommand` | TuiRunner callbacks | Animations, custom rendering, tick events |
| Declarative | `TuiAppCommand` | Element DSL | Dashboards, forms, data views |

## Installation

Add the `aesh-tamboui` dependency to your project. It is built as part of the aesh project under the `tamboui` Maven profile.

### Maven

```xml
<dependency>
    <groupId>org.aesh</groupId>
    <artifactId>aesh-tamboui</artifactId>
    <version>${aesh.version}</version>
</dependency>
```

The module depends on TamboUI artifacts (`tamboui-core`, `tamboui-widgets`, `tamboui-tui`, `tamboui-toolkit`, `tamboui-aesh-backend`). These are pulled in transitively.

## Quick Start

The simplest way to create a TUI command is to extend `TuiAppCommand` and implement `render()`:

```java
@CommandDefinition(name = "hello", description = "Show a hello panel")
public class HelloCommand extends TuiAppCommand {

    @Override
    protected Element render() {
        return panel("Hello TamboUI!",
                text("Welcome to aesh + TamboUI.\n\nPress 'q' to quit.")
        ).rounded().borderColor(Color.CYAN).fill();
    }
}
```

Register it like any other aesh command:

```java
AeshConsoleRunner.builder()
        .command(HelloCommand.class)
        .addExitCommand()
        .prompt("demo> ")
        .start();
```

When the user types `hello`, the TUI takes over the terminal. Press `q` or `Ctrl+C` to return to the aesh prompt.

## TuiAppCommand — Declarative UI

`TuiAppCommand` uses TamboUI's ToolkitRunner and Element DSL for declarative rendering. Override `render()` to return a UI element tree that is drawn each frame.

```java
@CommandDefinition(name = "status", description = "System status")
public class StatusCommand extends TuiAppCommand {

    @Override
    protected Element render() {
        return column(
                panel("System Status",
                        row(
                                gauge(72).label("CPU: 72%").gaugeColor(Color.RED)
                                        .title("CPU").rounded().fill(),
                                gauge(45).label("Mem: 45%").gaugeColor(Color.GREEN)
                                        .title("Memory").rounded().fill()
                        ).fill()
                ).rounded().borderColor(Color.BLUE).fill(),
                text("Press 'q' to quit").dim()
        ).fill();
    }
}
```

The Element DSL is available via `import static dev.tamboui.toolkit.Toolkit.*` and provides functions like `panel()`, `text()`, `column()`, `row()`, `gauge()`, `table()`, `tabs()`, `sparkline()`, `barChart()`, `calendar()`, `list()`, and `spacer()`.

### Key Handling

Override `onKeyEvent()` to handle keyboard input. The default implementation quits on `q` or `Ctrl+C`:

```java
@Override
protected boolean onKeyEvent(KeyEvent event, ToolkitRunner runner) {
    if (event.isQuit()) {
        runner.quit();
        return true;
    }
    if (event.isUp()) {
        tableState.selectPrevious();
        return true;
    }
    if (event.isDown()) {
        tableState.selectNext(totalItems);
        return true;
    }
    return false;
}
```

Return `true` if you handled the event, `false` to let it pass through.

### Initialization

Override `onStart(ToolkitRunner)` to run logic after the TUI starts, such as scheduling background data fetches:

```java
@Override
protected void onStart(ToolkitRunner runner) {
    // schedule initial data load, start timers, etc.
}
```

### Configuration

Override `configure(TuiConfig.Builder)` to adjust TUI settings like tick rate or mouse capture:

```java
@Override
protected TuiConfig.Builder configure(TuiConfig.Builder builder) {
    return builder.tickRate(Duration.ofMillis(200));
}
```

## TuiCommand — Event Loop

`TuiCommand` gives direct access to TamboUI's `TuiRunner` for event loop–based rendering. This is the right choice when you need tick events for animations, direct `Frame` rendering, or complex event handling.

```java
@CommandDefinition(name = "gauge", description = "Animated progress bar")
public class GaugeCommand extends TuiCommand {

    @Option(name = "speed", shortName = 's', defaultValue = {"100"},
            description = "Tick rate in milliseconds")
    private int speedMs;

    @Override
    protected TuiConfig.Builder configure(TuiConfig.Builder builder) {
        return builder.tickRate(Duration.ofMillis(speedMs));
    }

    @Override
    protected void runTui(TuiRunner runner, CommandInvocation invocation) throws Exception {
        AtomicInteger progress = new AtomicInteger(0);

        runner.run(
                (event, r) -> {
                    if (event instanceof KeyEvent) {
                        KeyEvent key = (KeyEvent) event;
                        if (key.isQuit()) {
                            r.quit();
                            return false;
                        }
                    }
                    if (event instanceof TickEvent) {
                        progress.getAndUpdate(v -> (v + 1) % 101);
                        return true;
                    }
                    return false;
                },
                frame -> {
                    Gauge gauge = Gauge.builder()
                            .percent(progress.get())
                            .label("Loading... " + progress.get() + "%")
                            .gaugeColor(Color.GREEN)
                            .block(Block.bordered())
                            .build();

                    frame.renderWidget(gauge, frame.area());
                }
        );
    }
}
```

The `runner.run()` method takes two lambdas:
- **Event handler** `(event, runner) -> boolean` — process events, return `true` to trigger a re-render
- **Renderer** `frame -> void` — draw widgets to the frame buffer

### Using Layouts with TuiRunner

For multi-widget layouts, use TamboUI's `Layout` to split the frame area:

```java
frame -> {
    Rect area = frame.area();
    List<Rect> rows = Layout.vertical()
            .constraints(
                    Constraint.length(3),   // fixed height for gauges
                    Constraint.fill(1),     // remaining space for chart
                    Constraint.length(1)    // status line
            )
            .split(area);

    frame.renderWidget(gauge, rows.get(0));
    frame.renderWidget(sparkline, rows.get(1));
    frame.renderWidget(statusLine, rows.get(2));
}
```

## TuiSupport — Manual Setup

For full control, use the `TuiSupport` utility class to create a backend or runner from any aesh `Shell`:

```java
// Inside a regular Command.execute():
Shell shell = invocation.getShell();

// Option A: get a pre-configured TuiConfig.Builder
TuiConfig config = TuiSupport.configBuilder(shell)
        .tickRate(Duration.ofMillis(100))
        .build();

// Option B: create a TuiRunner directly
try (TuiRunner runner = TuiSupport.createRunner(shell)) {
    runner.run(eventHandler, renderer);
}

// Option C: create a ToolkitRunner directly
try (ToolkitRunner runner = TuiSupport.createToolkitRunner(shell)) {
    runner.run(() -> myElement());
}
```

### TuiSupport API

| Method | Returns | Description |
|--------|---------|-------------|
| `createBackend(Shell)` | `AeshBackend` | Create a TamboUI backend from the shell's connection |
| `configBuilder(Shell)` | `TuiConfig.Builder` | Get a builder with backend pre-configured |
| `createRunner(Shell)` | `TuiRunner` | Create a TuiRunner with default config |
| `createRunner(Shell, TuiConfig.Builder)` | `TuiRunner` | Create a TuiRunner with custom config |
| `createToolkitRunner(Shell)` | `ToolkitRunner` | Create a ToolkitRunner with default config |

## Available Widgets

TamboUI provides a rich set of widgets you can use in your commands:

| Widget | Element DSL | Description |
|--------|-------------|-------------|
| Gauge | `gauge(percent)` | Progress bar with label and color |
| Sparkline | `sparkline(data)` | Compact line chart from data points |
| BarChart | `barChart()` | Grouped bar chart with colored bars |
| Table | `table()` | Data table with headers, column widths, and row selection |
| Tabs | `tabs(labels...)` | Tab bar with keyboard navigation |
| Calendar | `calendar(date)` | Month calendar with today highlight |
| List | `list(items...)` | Scrollable list with selection highlight |
| Paragraph | `text(content)` | Text block with styling |
| Panel | `panel(title, children)` | Bordered container with title |
| Layout | `row(...)`, `column(...)` | Horizontal and vertical layouts |

## Example: Interactive Table

A table with keyboard navigation using `TuiAppCommand`:

```java
@CommandDefinition(name = "employees", description = "Employee directory")
public class EmployeeCommand extends TuiAppCommand {

    private final TableState tableState = new TableState();

    @Override
    protected Element render() {
        return column(
                panel("Employee Directory",
                        table()
                                .header("ID", "Name", "Role", "City")
                                .widths(
                                        Constraint.length(4),
                                        Constraint.percentage(25),
                                        Constraint.percentage(25),
                                        Constraint.fill(1)
                                )
                                .row("1", "Alice", "Engineer", "San Francisco")
                                .row("2", "Bob", "Designer", "New York")
                                .row("3", "Carol", "Manager", "London")
                                .row("4", "Dave", "Analyst", "Berlin")
                                .row("5", "Eve", "Developer", "Tokyo")
                                .highlightStyle(Style.EMPTY.bg(Color.DARK_GRAY))
                                .highlightSymbol(">> ")
                                .state(tableState)
                                .fill()
                ).rounded().borderColor(Color.BLUE).fill(),
                text("Navigate: Up/Down | Quit: q").dim()
        ).fill();
    }

    @Override
    protected boolean onKeyEvent(KeyEvent event, ToolkitRunner runner) {
        if (event.isQuit()) {
            runner.quit();
            return true;
        }
        if (event.isUp()) {
            tableState.selectPrevious();
            return true;
        }
        if (event.isDown()) {
            tableState.selectNext(5); // total number of rows
            return true;
        }
        return false;
    }
}
```

## Example: Live Dashboard

A multi-panel dashboard combining gauges, sparkline, and dynamic data using `TuiCommand`:

```java
@CommandDefinition(name = "dashboard", description = "Live system dashboard")
public class DashboardCommand extends TuiCommand {

    @Override
    protected TuiConfig.Builder configure(TuiConfig.Builder builder) {
        return builder.tickRate(Duration.ofMillis(500));
    }

    @Override
    protected void runTui(TuiRunner runner, CommandInvocation invocation) throws Exception {
        Random rng = new Random();
        AtomicInteger cpu = new AtomicInteger(50);
        List<Long> cpuHistory = new ArrayList<>();

        runner.run(
                (event, r) -> {
                    if (event instanceof KeyEvent && ((KeyEvent) event).isQuit()) {
                        r.quit();
                        return false;
                    }
                    if (event instanceof TickEvent) {
                        cpu.set(clamp(cpu.get() + rng.nextInt(11) - 5, 0, 100));
                        cpuHistory.add((long) cpu.get());
                        if (cpuHistory.size() > 120) cpuHistory.remove(0);
                        return true;
                    }
                    return false;
                },
                frame -> {
                    Rect area = frame.area();
                    List<Rect> rows = Layout.vertical()
                            .constraints(
                                    Constraint.length(3),
                                    Constraint.fill(1)
                            )
                            .split(area);

                    // Top: gauge
                    Gauge gauge = Gauge.builder()
                            .percent(cpu.get())
                            .label("CPU " + cpu.get() + "%")
                            .gaugeColor(cpu.get() > 80 ? Color.RED : Color.GREEN)
                            .build();
                    frame.renderWidget(gauge, rows.get(0));

                    // Bottom: sparkline history
                    long[] histArr = cpuHistory.stream()
                            .mapToLong(Long::longValue).toArray();
                    Sparkline spark = Sparkline.builder()
                            .data(histArr)
                            .max(100)
                            .block(Block.builder().title("CPU History").build())
                            .style(Style.EMPTY.fg(Color.CYAN))
                            .build();
                    frame.renderWidget(spark, rows.get(1));
                }
        );
    }
}
```

## How It Works

The bridge between aesh and TamboUI is the `Connection` interface from aesh-readline:

```
┌─────────────────────────────────────┐
│      Your TUI Command               │
│   (TuiCommand / TuiAppCommand)      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│        aesh-tamboui Bridge           │
│  • TuiSupport, NonClosingConnection  │
│  • Shares Connection with TamboUI    │
└──────────────┬──────────────────────┘
               │
         ┌─────┴─────┐
         ▼           ▼
┌──────────────┐ ┌──────────────────┐
│  Æsh         │ │  TamboUI         │
│  • Terminal  │ │  • Widgets       │
│  • Commands  │ │  • Layouts       │
│  • Lifecycle │ │  • Event loop    │
└──────────────┘ └──────────────────┘
```

When a TUI command executes:

1. The command obtains aesh's `Connection` from `Shell.connection()`
2. The connection is wrapped in a `NonClosingConnection` to prevent TamboUI from closing it on exit
3. A TamboUI `AeshBackend` is created with the wrapped connection
4. TamboUI takes over the terminal for rendering and event handling
5. When the user quits, TamboUI cleans up and control returns to aesh

The `NonClosingConnection` wrapper is essential — without it, TamboUI's cleanup would close the terminal connection and terminate the aesh session.

## Running the Demo

The module includes a demo application with several example commands. Build the shaded jar and run it:

```bash
# Build with the tamboui profile
mvn package -Ptamboui -pl aesh-tamboui -am

# Run the demo
java -jar aesh-tamboui/target/aesh-tamboui-*.jar
```

Available demo commands:

| Command | Description |
|---------|-------------|
| `tui-hello` | Minimal styled panel |
| `tui-gauge` | Animated progress bar with configurable speed |
| `tui-table` | Interactive table with keyboard navigation |
| `tui-sparkline` | Live sparkline chart with random data |
| `tui-barchart` | Grouped bar chart showing server metrics |
| `tui-tabs` | Tabbed interface with different content per tab |
| `tui-calendar` | Month calendar with month navigation |
| `tui-dashboard` | Multi-panel live dashboard |

## See Also

- [Terminal Graphics](graphics) — lower-level drawing primitives built into aesh
- [Progress Bar](progress-bar) — simple progress bar utility built into aesh
- [Runners](runners) — aesh command execution modes
