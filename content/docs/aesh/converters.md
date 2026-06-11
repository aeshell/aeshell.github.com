---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Converters'
weight: 10
---

Converters transform the raw string input from the command line into the Java type your option or argument expects. They run automatically when Æsh populates your command object.

## Converter Interface

```java
public interface Converter<T, C extends ConverterInvocation> {
    T convert(C converterInvocation) throws OptionValidatorException;
}
```

The `ConverterInvocation` provides:
- `getInput()` -- the raw string value from the command line
- `getAeshContext()` -- the current Æsh context

## Built-in Converters

Æsh converts these types automatically -- no converter needed:

- `String`
- `int` / `Integer`, `long` / `Long`, `short` / `Short`, `byte` / `Byte`
- `float` / `Float`, `double` / `Double`
- `boolean` / `Boolean` (accepts `true`/`false`, `yes`/`no`, `1`/`0` — case-insensitive; rejects unrecognized values)
- `char` / `Character`
- `File`, `Path`, [`Resource`](../resources)
- Any `Enum` type (matched by name, case-insensitive)

## Enum Conversion

Enums work out of the box:

```java
public enum LogLevel {
    DEBUG, INFO, WARN, ERROR
}

@CommandDefinition(name = "log", description = "Configure logging")
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

```
$ log --level DEBUG
Log level: DEBUG
```

Tab completion for enum values is also automatic.

## Custom Converters

Write a converter when you need a type that isn't built in. Implement `Converter<T, ConverterInvocation>`:

```java
import org.aesh.command.converter.Converter;
import org.aesh.command.converter.ConverterInvocation;
import org.aesh.command.validator.OptionValidatorException;

import java.time.Duration;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class DurationConverter implements Converter<Duration, ConverterInvocation> {

    private static final Pattern PATTERN = Pattern.compile("(\\d+)([smhd])");

    @Override
    public Duration convert(ConverterInvocation invocation) throws OptionValidatorException {
        String input = invocation.getInput();
        Matcher matcher = PATTERN.matcher(input.toLowerCase());

        if (!matcher.matches()) {
            throw new OptionValidatorException("Invalid duration: " + input +
                ". Use format like: 30s, 5m, 2h, 1d");
        }

        long value = Long.parseLong(matcher.group(1));
        switch (matcher.group(2)) {
            case "s": return Duration.ofSeconds(value);
            case "m": return Duration.ofMinutes(value);
            case "h": return Duration.ofHours(value);
            case "d": return Duration.ofDays(value);
            default:  throw new OptionValidatorException("Unknown unit: " + matcher.group(2));
        }
    }
}
```

Use it with `@Option`:

```java
@CommandDefinition(name = "timeout", description = "Set a timeout")
public class TimeoutCommand implements Command<CommandInvocation> {

    @Option(
        name = "duration",
        converter = DurationConverter.class,
        description = "Timeout duration (e.g., 30s, 5m, 2h)"
    )
    private Duration duration;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Timeout set to " + duration.toSeconds() + " seconds");
        return CommandResult.SUCCESS;
    }
}
```

```
$ timeout --duration 5m
Timeout set to 300 seconds
```

## Error Handling

When conversion fails, throw `OptionValidatorException` with a message describing what went wrong and what the user should provide instead. Aesh displays the message to the user and aborts command parsing:

```
$ timeout --duration abc
Invalid duration: abc. Use format like: 30s, 5m, 2h, 1d
```

All built-in converters validate their input and throw `OptionValidatorException` with descriptive messages:

```
$ myapp --port abc
Invalid integer value: 'abc'

$ myapp --fresh=foobar
Invalid boolean value: 'foobar'. Expected: true/false, yes/no, 1/0

$ myapp --size 999999999999999999999
Invalid long value: '999999999999999999999'
```

## Converter vs Validator

- **Converters** transform the input string into a typed value. They answer: *"What does this string mean?"*
- **Validators** check whether a value is acceptable. They answer: *"Is this value allowed?"*

Converters run first. If conversion succeeds, validators run next. Use a converter when you need a custom type; use a [Validator](../validators) when the type is standard but the allowed values are restricted (e.g., port must be 1-65535).
