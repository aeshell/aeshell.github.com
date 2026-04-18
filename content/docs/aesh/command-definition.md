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
| `defaultValueProvider` | `Class<? extends DefaultValueProvider>` | `NullDefaultValueProvider.class` | Dynamic default value resolver |
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

## Dynamic Default Value Provider

The `defaultValueProvider` attribute specifies a class that resolves option defaults at runtime. This is useful when defaults come from configuration files, environment variables, or other external sources not known at compile time.

### Implementing a Provider

Create a class that implements `DefaultValueProvider`:

```java
public class ConfigDefaultProvider implements DefaultValueProvider {

    @Override
    public String defaultValue(ProcessedOption option) {
        // Use option.name() and option.parent().name() to build a config key
        String key = option.parent().name() + "." + option.name();
        return Configuration.get(key);  // returns null if not configured
    }
}
```

### Registering the Provider

```java
@CommandDefinition(
    name = "init",
    description = "Initialize a project",
    defaultValueProvider = ConfigDefaultProvider.class
)
public class InitCommand implements Command<CommandInvocation> {

    @Option(defaultValue = "hello")
    private String template;

    @Option
    private String editor;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Template: " + template);
        invocation.println("Editor: " + editor);
        return CommandResult.SUCCESS;
    }
}
```

### Value Precedence

When determining option values, the precedence is (highest to lowest):

1. **User-provided value** -- Explicitly set on the command line
2. **Dynamic default** -- Returned by the `DefaultValueProvider` (if non-null)
3. **Static default** -- The `defaultValue` from the annotation
4. **null** -- If nothing else is set

If the provider returns `null` for an option, aesh falls back to the static `defaultValue`. This lets you use annotation defaults as fallbacks:

```java
// Provider returns "from-config" for template -> uses "from-config"
// Provider returns null for editor -> falls back to static default "vi"
@Option(defaultValue = "vi")
private String editor;
```

See [Options - Dynamic Default Values](/docs/aesh/options#dynamic-default-values) for more details.

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
