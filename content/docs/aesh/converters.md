---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Converters'
weight: 10
---

Converters transform string input into custom types.

## Converter Interface

Implement the `Converter<T>` interface:

```java
public interface Converter<T> {
    T convert(String input) throws ConverterException;
}
```

## Custom Type Conversion

Convert a string to a custom class:

```java
public class DurationConverter implements Converter<Duration> {

    @Override
    public Duration convert(String input) throws ConverterException {
        try {
            String[] parts = input.split("(?<=\\D)(?=\\d)");
            long value = Long.parseLong(parts[0]);
            String unit = parts[1].toLowerCase();
            
            switch (unit) {
                case "s": return Duration.ofSeconds(value);
                case "m": return Duration.ofMinutes(value);
                case "h": return Duration.ofHours(value);
                default: throw new ConverterException("Invalid unit: " + unit);
            }
        } catch (Exception e) {
            throw new ConverterException("Invalid duration: " + input + 
                ". Use format like: 30s, 5m, 2h");
        }
    }
}
```

Usage:

```java
@CommandDefinition(name = "timeout")
public class TimeoutCommand implements Command<CommandInvocation> {

    @Option(
        name = "duration",
        converter = DurationConverter.class,
        description = "Timeout duration (e.g., 30s, 5m, 2h)"
    )
    private Duration duration;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Timeout: " + duration.toSeconds() + " seconds");
        return CommandResult.SUCCESS;
    }
}
```

Usage: `timeout --duration 5m`

## Built-in Converters

Æsh includes converters for common types:
- `String`
- `int`, `Integer`
- `long`, `Long`
- `float`, `Float`
- `double`, `Double`
- `boolean`, `Boolean`
- `File`
- `Path`

## Enum Conversion

Enums are automatically converted:

```java
public enum LogLevel {
    DEBUG, INFO, WARN, ERROR
}

@CommandDefinition(name = "log")
public class LogCommand implements Command<CommandInvocation> {

    @Option(description = "Log level")
    private LogLevel level = LogLevel.INFO;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Log level: " + level);
        return CommandResult.SUCCESS;
    }
}
```

Usage: `log --level DEBUG`
