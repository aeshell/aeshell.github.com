---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Options'
weight: 4
---

The `@Option` annotation defines command-line options (flags with values).

## Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `name` | `String` | `""` | Option name (variable name if empty) |
| `shortName` | `char` | `'\u0000'` | Short name (e.g., `-v`) |
| `description` | `String` | `""` | Help description |
| `argument` | `String` | `""` | Value type description |
| `required` | `boolean` | `false` | Is option required? |
| `hasValue` | `boolean` | `true` | Does option accept a value? |
| `defaultValue` | `String[]` | `{}` | Default values |
| `askIfNotSet` | `boolean` | `false` | Prompt user if not set |
| `overrideRequired` | `boolean` | `false` | Override required validation |
| `acceptNameWithoutDashes` | `boolean` | `false` | Allow option name without `--` prefix |
| `converter` | `Class<? extends Converter>` | `NullConverter.class` | Custom value converter |
| `completer` | `Class<? extends OptionCompleter>` | `NullOptionCompleter.class` | Custom completer |
| `validator` | `Class<? extends OptionValidator>` | `NullValidator.class` | Custom validator |
| `activator` | `Class<? extends OptionActivator>` | `NullActivator.class` | Custom activator |
| `renderer` | `Class<? extends OptionRenderer>` | `NullOptionRenderer.class` | Custom renderer |
| `parser` | `Class<? extends OptionParser>` | `AeshOptionParser.class` | Custom parser |

## Basic Example

```java
@CommandDefinition(name = "greet")
public class GreetCommand implements Command<CommandInvocation> {

    @Option(shortName = 'n', description = "Name to greet")
    private String name;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Hello, " + name + "!");
        return CommandResult.SUCCESS;
    }
}
```

Usage: `greet --name Alice` or `greet -n Alice`

## Boolean Flags

For boolean fields, `hasValue` can be `false`:

```java
@Option(shortName = 'v', hasValue = false, description = "Verbose output")
private boolean verbose;
```

Usage: `greet -v` or `greet --verbose`

## Required Options

```java
@Option(required = true, description = "Required file path")
private String filePath;
```

## Default Values

```java
@Option(defaultValue = "INFO", description = "Log level")
private String logLevel;
```

## Custom Types with Converter

```java
@Option(converter = PathConverter.class, description = "Directory path")
private Path directory;

public static class PathConverter implements Converter<Path> {
    @Override
    public Path convert(String input) {
        return Paths.get(input);
    }
}
```

## Short Names

The `shortName` defines the single-character option:

```java
@Option(name = "verbose", shortName = 'v', description = "Verbose mode")
private boolean verbose;
```

Both `--verbose` and `-v` work.
