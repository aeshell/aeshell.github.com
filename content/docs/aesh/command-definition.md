---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Command Definition'
weight: 3
---

The `@CommandDefinition` annotation is used to define a command class.

## Required Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | `String` | The command name |

## Optional Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `aliases` | `String[]` | `{}` | Alternative names for the command |
| `description` | `String` | `""` | Command description shown in help |
| `generateHelp` | `boolean` | `false` | Auto-generate `--help` option |
| `disableParsing` | `boolean` | `false` | Skip parsing (everything goes to @Arguments) |
| `version` | `String` | `""` | Version string (adds `--version`, `-v` option) |
| `validator` | `Class<? extends CommandValidator>` | `NullCommandValidator.class` | Validator to run before execution |
| `resultHandler` | `Class<? extends ResultHandler>` | `NullResultHandler.class` | Handler to run after execution |
| `activator` | `Class<? extends CommandActivator>` | `NullCommandActivator.class` | Activator to check if command is available |

## Example

```java
@CommandDefinition(
    name = "copy",
    aliases = {"cp"},
    description = "Copy files",
    generateHelp = true
)
public class CopyCommand implements Command<CommandInvocation> {
    
    @Option(shortName = 'r', description = "Recursive copy")
    private boolean recursive;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // implementation
        return CommandResult.SUCCESS;
    }
}
```

## Command Interface

Your command must implement the `Command<T extends CommandInvocation>` interface:

```java
public interface Command<T extends CommandInvocation> {
    CommandResult execute(T commandInvocation) 
        throws CommandException, InterruptedException;
}
```

### CommandResult

Return one of the following:
- `CommandResult.SUCCESS` - Command completed successfully
- `CommandResult.FAILURE` - Command failed
- `CommandResult.RETURN` - Return from current subcommand

### CommandInvocation

Provides access to:
- `println(String)` - Output text to the console
- `print(String)` - Output text without newline
- `stop()` - Stop the console
- `getShell()` - Access the shell
- `getHelpInfo(String)` - Get help text
