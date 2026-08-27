---
date: '2026-06-18T15:00:00+01:00'
draft: false
title: 'Charts'
weight: 22
---

The `aesh-charts` module provides terminal charting capabilities using Unicode block and braille characters. Charts render directly to the terminal without requiring a GUI or external tools.

## Installation

Add the `aesh-charts` dependency alongside `aesh`:

```xml
<dependency>
    <groupId>org.aesh</groupId>
    <artifactId>aesh-charts</artifactId>
    <version>3.17.3</version>
</dependency>
```

Gradle:

```groovy
implementation 'org.aesh:aesh-charts:3.17.3'
```

## Chart Types

### Sparkline

A compact inline chart showing a data series as a single row of block characters.

```java
import org.aesh.charts.Sparkline;
import org.aesh.charts.Canvas;

double[] data = {1, 3, 5, 2, 8, 4, 6, 3, 7, 2};
Sparkline sparkline = new Sparkline(data);
Canvas canvas = sparkline.render(40, 1); // width, height
System.out.println(canvas.toString());
```

### Bar Chart

Vertical or horizontal bar charts for comparing categorical data.

```java
import org.aesh.charts.BarChart;
import org.aesh.charts.Canvas;

BarChart chart = BarChart.builder()
        .title("Sales by Region")
        .bar("North", 45)
        .bar("South", 32)
        .bar("East", 58)
        .bar("West", 41)
        .build();

Canvas canvas = chart.render(60, 15);
System.out.println(canvas.toString());
```

### Line Chart

Line charts using braille characters for high-resolution rendering in the terminal.

```java
import org.aesh.charts.LineChart;
import org.aesh.charts.Canvas;

double[] temperatures = {18, 20, 22, 25, 27, 26, 23, 20, 18, 16};
LineChart chart = LineChart.builder()
        .title("Temperature")
        .series("Temp", temperatures)
        .build();

Canvas canvas = chart.render(60, 15);
System.out.println(canvas.toString());
```

Multi-series charts are supported:

```java
LineChart chart = LineChart.builder()
        .title("CPU vs Memory")
        .series("CPU", cpuData)
        .series("Memory", memData)
        .build();
```

### Time Series Chart

Line chart optimized for time-series data with automatic X-axis labeling.

```java
import org.aesh.charts.TimeSeriesChart;

TimeSeriesChart chart = TimeSeriesChart.builder()
        .title("Request Latency")
        .series("p99", latencyData)
        .build();
```

### Multi-Plot

Combine multiple charts into a stacked dashboard.

```java
import org.aesh.charts.MultiPlot;

MultiPlot plot = MultiPlot.builder()
        .add(cpuChart)
        .add(memoryChart)
        .add(networkChart)
        .build();

Canvas canvas = plot.render(80, 40);
System.out.println(canvas.toString());
```

## Canvas and Encoding

Charts render to a `Canvas` object which can be converted to a string. Two encoding modes are available:

- **BrailleEncoder** -- uses Unicode braille characters (U+2800-U+28FF) for high-resolution point plotting. Each character cell represents a 2x4 grid of dots.
- **BlockEncoder** -- uses Unicode block elements (full block, half blocks, quarter blocks) for bar charts and area fills.

## Markers

Add visual markers to charts for annotations:

```java
import org.aesh.charts.Marker;
import org.aesh.charts.HorizontalLine;

LineChart chart = LineChart.builder()
        .title("CPU Usage")
        .series("cpu", data)
        .marker(new HorizontalLine(80, "threshold"))
        .marker(new Marker(5, data[5], "peak"))
        .build();
```

## Using Charts in Commands

Charts integrate naturally with aesh commands:

```java
@CommandDefinition(name = "stats", description = "Show system stats")
public class StatsCommand implements Command<CommandInvocation> {

    @Override
    public CommandResult execute(CommandInvocation ci) {
        double[] cpu = getCpuHistory();
        LineChart chart = LineChart.builder()
                .title("CPU Usage (%)")
                .series("cpu", cpu)
                .build();

        Size size = ci.getShell().size();
        Canvas canvas = chart.render(size.getWidth(), 15);
        ci.println(canvas.toString());
        return CommandResult.SUCCESS;
    }
}
```

## Interactive Example

The `examples` module includes a `ChartExample` with 6 demo commands:

```bash
mvn -Pexamples exec:java -pl examples -Dexec.mainClass=examples.ChartExample
```
