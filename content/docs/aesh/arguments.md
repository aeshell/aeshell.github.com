---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Arguments'
weight: 5
---

The `@Argument` annotation defines positional command-line arguments.

## Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `description` | `String` | `""` | Help description |
| `required` | `boolean` | `false` | Is argument required? |
| `defaultValue` | `String[]` | `{}` | Default values |
| `askIfNotSet` | `boolean` | `false` | Prompt user if not set |
| `overrideRequired` | `boolean` | `false` | Override required validation |
| `inherited` | `boolean` | `false` | Make argument available to subcommands in sub-command mode |
| `converter` | `Class<? extends Converter>` | `NullConverter.class` | Custom value converter |
| `completer` | `Class<? extends OptionCompleter>` | `NullOptionCompleter.class` | Custom completer |
| `validator` | `Class<? extends OptionValidator>` | `NullValidator.class` | Custom validator |
| `activator` | `Class<? extends OptionActivator>` | `NullActivator.class` | Custom activator |
| `renderer` | `Class<? extends OptionRenderer>` | `NullOptionRenderer.class` | Custom renderer |
| `parser` | `Class<? extends OptionParser>` | `AeshOptionParser.class` | Custom parser |

## Basic Example

```java
@CommandDefinition(name = "echo")
public class EchoCommand implements Command<CommandInvocation> {

    @Argument(description = "Text to echo")
    private String text;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println(text);
        return CommandResult.SUCCESS;
    }
}
```

Usage: `echo Hello World`

## Multiple Arguments with @Arguments

For multiple values, use `@Arguments` (plural) with a `Collection`:

```java
@CommandDefinition(name = "sum")
public class SumCommand implements Command<CommandInvocation> {

    @Arguments(description = "Numbers to sum")
    private List<Integer> numbers;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        int sum = numbers.stream().mapToInt(Integer::intValue).sum();
        invocation.println("Sum: " + sum);
        return CommandResult.SUCCESS;
    }
}
```

Usage: `sum 1 2 3 4 5`

## Required Argument

```java
@Argument(required = true, description = "Required file")
private String file;
```

## Default Value

```java
@Argument(defaultValue = "output.txt", description = "Output file")
private String outputFile;
```

## Combined with Options

Arguments and options can be used together:

```java
@CommandDefinition(name = "copy")
public class CopyCommand implements Command<CommandInvocation> {

    @Argument(required = true, description = "Source file")
    private String source;

    @Option(shortName = 'd', description = "Destination directory")
    private String destination = ".";

    @Option(shortName = 'v', hasValue = false, description = "Verbose")
    private boolean verbose;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // implementation
        return CommandResult.SUCCESS;
    }
}
```

Usage: `copy myfile.txt -d /tmp -v`

## Inherited Arguments

Inherited arguments are automatically available to all subcommands when using [sub-command mode](/docs/aesh/sub-command-mode). Mark an argument with `inherited = true` on a group command, and subcommands with matching field names will have the value auto-populated.

```java
@GroupCommandDefinition(
    name = "project",
    groupCommands = {BuildCommand.class, TestCommand.class}
)
public class ProjectCommand implements Command<CommandInvocation> {

    @Argument(description = "Project name", inherited = true)
    private String projectName;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.enterSubCommandMode(this);
        return CommandResult.SUCCESS;
    }
}

@CommandDefinition(name = "build")
public class BuildCommand implements Command<CommandInvocation> {

    // This field is auto-populated from parent's inherited argument
    @Argument(description = "Project name")
    private String projectName;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Building " + projectName);
        return CommandResult.SUCCESS;
    }
}
```

You can also access inherited values programmatically:

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    String name = invocation.getInheritedValue("projectName", String.class);
    invocation.println("Project: " + name);
    return CommandResult.SUCCESS;
}
```

See [Sub-Command Mode](/docs/aesh/sub-command-mode) for complete documentation on inherited options and arguments.
