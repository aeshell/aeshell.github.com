---
date: '2026-04-19T13:00:00+02:00'
draft: false
title: 'Mixins'
weight: 7
---

The `@Mixin` annotation enables composition-based reuse of option groups across commands without requiring class inheritance. Define shared options in a plain Java class, then include them in any command by annotating a field with `@Mixin`.

## Why Mixins?

When multiple commands share the same options (logging, output format, connection settings), you have two choices:

1. **Inheritance** -- Put shared options in a base class and extend it. This works but forces a single inheritance chain and couples commands together.
2. **Mixins** -- Define shared options in a standalone class and compose them into any command. Commands stay independent and can mix in multiple option groups.

Mixins are the better choice when:
- Commands already extend different base classes
- You want to compose multiple independent option groups
- The shared options don't imply an "is-a" relationship

## Basic Example

Define a mixin class with annotated fields (no special interface needed):

```java
public class LoggingMixin {

    @Option(name = "verbose", shortName = 'v', hasValue = false,
            description = "Enable verbose output")
    boolean verbose;

    @Option(name = "log-level", defaultValue = "INFO",
            description = "Log level")
    String logLevel;
}
```

Use it in a command with `@Mixin`:

```java
@CommandDefinition(name = "deploy", description = "Deploy application")
public class DeployCommand implements Command<CommandInvocation> {

    @Mixin
    LoggingMixin logging;

    @Option(description = "Target environment")
    private String environment;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (logging.verbose) {
            invocation.println("[VERBOSE] Deploying to " + environment);
        }
        invocation.println("Deployed to " + environment);
        return CommandResult.SUCCESS;
    }
}
```

Usage:
```bash
$ deploy --environment prod --verbose --log-level DEBUG
[VERBOSE] Deploying to prod
Deployed to prod
```

The mixin's options (`--verbose`, `--log-level`) appear as regular options on the command. Users don't see any difference -- the options are flattened into the command's option list.

## Multiple Mixins

A command can include multiple mixins:

```java
public class OutputMixin {
    @Option(name = "format", defaultValue = "text",
            description = "Output format (text, json, yaml)")
    String format;

    @Option(name = "quiet", shortName = 'q', hasValue = false,
            description = "Suppress non-essential output")
    boolean quiet;
}

@CommandDefinition(name = "status", description = "Show status")
public class StatusCommand implements Command<CommandInvocation> {

    @Mixin
    LoggingMixin logging;

    @Mixin
    OutputMixin output;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (!output.quiet) {
            invocation.println("Status: OK (format=" + output.format + ")");
        }
        return CommandResult.SUCCESS;
    }
}
```

Usage: `status --verbose --format json --quiet`

## Supported Annotations

Mixin classes support all option and argument annotations:

- `@Option` -- Single-value options
- `@OptionList` -- Multi-value list options
- `@OptionGroup` -- Key-value map options
- `@Argument` -- Positional argument
- `@Arguments` -- Multiple positional arguments

```java
public class ConnectionMixin {

    @Option(name = "host", shortName = 'h', defaultValue = "${DB_HOST:localhost}",
            description = "Server hostname")
    String host;

    @Option(name = "port", shortName = 'p', defaultValue = "${DB_PORT:5432}",
            description = "Server port")
    int port;

    @OptionList(name = "tags", description = "Connection tags")
    List<String> tags;

    @OptionGroup(shortName = 'D', description = "Connection properties")
    Map<String, String> properties;
}
```

All option features work within mixins: custom converters, completers, validators, activators, renderers, default values (including environment variables), `required`, `askIfNotSet`, `negatable`, and `optionalValue`.

## Mixin Inheritance

Mixin classes can extend other classes. Options from the mixin's superclass chain are included:

```java
public class BaseMixin {
    @Option(name = "debug", hasValue = false, description = "Debug mode")
    boolean debug;
}

public class LoggingMixin extends BaseMixin {
    @Option(name = "verbose", hasValue = false, description = "Verbose output")
    boolean verbose;
}

// Command gets both --debug and --verbose
@CommandDefinition(name = "run", description = "Run application")
public class RunCommand implements Command<CommandInvocation> {
    @Mixin
    LoggingMixin logging;
    // ...
}
```

## Field Initialization

Mixin fields can be pre-initialized or left null. If the mixin field is null when parsing begins, Aesh automatically creates an instance using the no-arg constructor:

```java
// Both work:
@Mixin
LoggingMixin logging;              // auto-created on first use

@Mixin
LoggingMixin logging = new LoggingMixin();  // pre-initialized
```

Pre-initialization is useful when you want to set defaults programmatically.

## How It Works

When Aesh processes a command class, it detects `@Mixin` fields and collects their annotated options into the command's option list. At parse time:

1. Option values are parsed from the command line as usual
2. For mixin options, values are injected into the mixin object (not the command object)
3. The mixin object is accessible through the annotated field on the command

The mixin object is automatically created if null, so you can always access mixin fields in your `execute()` method.

## Annotation Processor Support

The [annotation processor](annotation-processor) fully supports `@Mixin`. When using `aesh-processor`, mixin options are resolved at compile time and included in the generated metadata. Runtime reflection is only used for field injection into the mixin object (same as for private fields on regular commands).

## Mixins vs Inherited Options

Both mixins and [inherited options](options#inherited-options) share options across commands, but they solve different problems:

| | Mixins | Inherited Options |
|---|---|---|
| **Scope** | Any command, independently | Parent to child in a group |
| **Access** | Direct field access on the mixin | Field matching or `@ParentCommand` |
| **Multiplicity** | Multiple mixins per command | One parent per child |
| **Use case** | Cross-cutting concerns (logging, output format) | Group-wide settings (config file, verbose) |

Use mixins when the shared options are a reusable concern that applies across unrelated commands. Use inherited options when a parent command defines settings that its subcommands should inherit.
