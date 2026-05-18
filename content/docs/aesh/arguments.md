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
| `paramLabel` | `String` | `""` | Display label in help synopsis (defaults to field name) |
| `arity` | `String` | `""` | Number of values accepted (e.g., `"0..1"`, `"1"`) |
| `index` | `String` | `"0"` | Positional index or range (e.g., `"0"`, `"2..2"`) |
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

The `@Arguments` annotation is for multiple positional values using a `Collection`:

### @Arguments Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `description` | `String` | `""` | Help description |
| `paramLabel` | `String` | `""` | Display label in help synopsis (defaults to field name) |
| `arity` | `String` | `""` | Number of values accepted (e.g., `"2"`, `"1..*"`, `"0..1"`) |
| `index` | `String` | `"1..*"` | Positional index or range (e.g., `"1..*"`, `"3..5"`) |
| `type` | `Class<?>` | `String.class` | Element type |
| `valueSeparator` | `char` | `' '` | Separator between values |
| `required` | `boolean` | `false` | Is at least one value required? |
| `defaultValue` | `String[]` | `{}` | Default values |
| `askIfNotSet` | `boolean` | `false` | Prompt user if not set |
| `converter` | `Class<? extends Converter>` | `NullConverter.class` | Custom value converter |
| `completer` | `Class<? extends OptionCompleter>` | `NullOptionCompleter.class` | Custom completer |
| `validator` | `Class<? extends OptionValidator>` | `NullValidator.class` | Custom validator |
| `activator` | `Class<? extends OptionActivator>` | `NullActivator.class` | Custom activator |
| `parser` | `Class<? extends OptionParser>` | `AeshOptionParser.class` | Custom parser |

### Basic Example

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

## Indexed Positional Parameters

Aesh supports explicit positional indexes on both `@Argument` and `@Arguments`.

### Index Syntax

| Syntax | Meaning |
|--------|---------|
| `"0"` | Exactly positional index 0 |
| `"2..4"` | Positional indexes 2 through 4 |
| `"1..*"` | Positional indexes 1 and above |

### Multiple `@Argument` Example

```java
@CommandDefinition(name = "copy2")
public class Copy2Command implements Command<CommandInvocation> {

    @Argument(index = "0", description = "Source path")
    private String source;

    @Argument(index = "1", description = "Destination path")
    private String destination;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println(source + " -> " + destination);
        return CommandResult.SUCCESS;
    }
}
```

### Combining `@Argument` with `@Arguments`

`@Argument` (singular) and `@Arguments` (plural) can be used together with explicit indexes. A common pattern is one singular argument at index `0` and overflow values from index `1` onward:

```java
@CommandDefinition(name = "run", description = "Run a script")
public class RunCommand implements Command<CommandInvocation> {

    @Argument(index = "0", description = "Script file or URL")
    private String scriptFile;

    @Arguments(index = "1..*", description = "Script arguments")
    private List<String> scriptArgs = new ArrayList<>();

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Script: " + scriptFile);
        invocation.println("Args: " + scriptArgs);
        return CommandResult.SUCCESS;
    }
}
```

Usage:
```bash
$ run myscript.java arg1 arg2
Script: myscript.java
Args: [arg1, arg2]

$ run myscript.java
Script: myscript.java
Args: []
```

When only `@Argument` or `@Arguments` is defined, default indexes preserve legacy behavior.

### Validation Rules

- Positional ranges must not overlap.
- If a token lands on an unsupported positional index, parsing fails with an error that includes the token index and declared ranges.

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

## Param Label

By default, the help synopsis shows the field name (e.g., `<input>`). Use `paramLabel` to customize the display label:

```java
@Argument(paramLabel = "scriptOrFile", description = "A file or URL to run")
private String input;
```

Help shows: `Usage: run [<options>] <scriptOrFile>` instead of `Usage: run [<options>] <input>`.

`paramLabel` is also available on `@Arguments`:

```java
@Arguments(paramLabel = "sources", description = "Source files to compile")
private List<String> files;
```

## Arity

The `arity` attribute controls how many values an argument accepts. By default, `@Argument` accepts 0 or 1 values, and `@Arguments` accepts 0 or more. Use `arity` to override this.

### Syntax

| Arity | Meaning |
|-------|---------|
| `"2"` | Exactly 2 values |
| `"0..1"` | Optional (0 or 1) |
| `"1..*"` | One or more |
| `"0..*"` | Zero or more (default for `@Arguments`) |
| `"2..4"` | Between 2 and 4 |

### Example: Exact Count

```java
@CommandDefinition(name = "config-set", description = "Set a configuration value")
public class ConfigSetCommand implements Command<CommandInvocation> {

    @Arguments(arity = "2", paramLabel = "args", description = "key and value")
    private List<String> args;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        String key = args.get(0);
        String value = args.get(1);
        invocation.println("Set " + key + " = " + value);
        return CommandResult.SUCCESS;
    }
}
```

```bash
$ config-set mykey myvalue    # OK
$ config-set mykey            # Error: Argument 'args' requires at least 2 values, but got 1.
$ config-set a b c            # Error: Too many arguments. Maximum is 2.
```

Help synopsis: `Usage: config-set [<options>] <args> <args>`

### Example: One or More

```java
@Arguments(arity = "1..*", paramLabel = "files", description = "Files to process")
private List<String> files;
```

Help synopsis: `Usage: process [<options>] <files>...`

### Example: Optional Single

```java
@Arguments(arity = "0..1", description = "Optional output file")
private List<String> output;
```

Help synopsis: `Usage: generate [<options>] [<output>]`

### Interaction with `required`

When `arity` is set, it takes precedence over `required` for count validation. Setting `arity = "1..*"` implicitly requires at least one value. Setting `arity = "0..1"` makes the argument optional regardless of the `required` flag.

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

## Passthrough Arguments

When a command needs to forward arguments to another process (e.g., a script runner), use `stopAtFirstPositional = true` on `@CommandDefinition`. After the first positional argument, all remaining tokens -- even those that look like options -- are collected as arguments:

```java
@CommandDefinition(
    name = "run",
    description = "Run a script",
    stopAtFirstPositional = true,
    generateHelp = true
)
public class RunCommand implements Command<CommandInvocation> {

    @Option(hasValue = false, description = "Verbose")
    private boolean verbose;

    @Argument(description = "Script file")
    private String script;

    @Arguments
    private List<String> scriptArgs;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // script and scriptArgs are split automatically
        return CommandResult.SUCCESS;
    }
}
```

```
$ run --verbose myscript.java -Dfoo=bar --help
```

Here `--verbose` is parsed as a command option, `myscript.java` is the first positional argument, and `-Dfoo=bar --help` are passthrough arguments collected in `args`.

See [Command Definition](/docs/aesh/command-definition#stop-at-first-positional) for more details.

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
