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
| `stopAtFirstPositional` | `boolean` | `false` | Stop option parsing after the first positional argument |
| `helpUrl` | `String` | `""` | URL to documentation (shown in `--help` output) |

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

## Stop at First Positional

When `stopAtFirstPositional = true`, option parsing stops as soon as the first positional argument is consumed. All remaining tokens are treated as positional arguments, even if they look like options.

This is useful for commands that pass arguments through to another process:

```java
@CommandDefinition(
    name = "run",
    description = "Run a script",
    stopAtFirstPositional = true,
    generateHelp = true
)
public class RunCommand implements Command<CommandInvocation> {

    @Option(hasValue = false, description = "Enable verbose output")
    private boolean verbose;

    @Arguments
    private List<String> args;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // args contains the script name and all arguments after it
        return CommandResult.SUCCESS;
    }
}
```

With the command above:

| Input | Result |
|-------|--------|
| `run --verbose myscript.java` | `verbose=true`, args=`[myscript.java]` |
| `run --verbose myscript.java -Dfoo=bar --help` | `verbose=true`, args=`[myscript.java, -Dfoo=bar, --help]` |
| `run myscript.java --verbose` | `verbose=false`, args=`[myscript.java, --verbose]` |
| `run --help` | Displays help output |

Note that `--help` (and `--version`) before the first positional still work normally when `generateHelp = true`. Only tokens after the first positional are treated as passthrough arguments.

### CommandInvocation

Provides access to:
- `println(String)` - Output text to the console
- `print(String)` - Output text without newline
- `stop()` - Stop the console
- `getShell()` - Access the shell
- `getHelpInfo(String)` - Get help text
