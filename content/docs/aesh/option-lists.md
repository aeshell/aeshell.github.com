---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Option Lists'
weight: 6
---

The `@OptionList` annotation defines options that accept multiple values separated by a delimiter.

## Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `name` | `String` | `""` | Option name (variable name if empty) |
| `shortName` | `char` | `'\u0000'` | Short name (e.g., `-p`) |
| `description` | `String` | `""` | Help description |
| `valueSeparator` | `char` | `','` | Character separating values |
| `required` | `boolean` | `false` | Is option required? |
| `defaultValue` | `String[]` | `{}` | Default values |
| `askIfNotSet` | `boolean` | `false` | Prompt user if not set |
| `converter` | `Class<? extends Converter>` | `NullConverter.class` | Custom value converter |
| `completer` | `Class<? extends OptionCompleter>` | `NullOptionCompleter.class` | Custom completer |
| `validator` | `Class<? extends OptionValidator>` | `NullValidator.class` | Custom validator |
| `activator` | `Class<? extends OptionActivator>` | `NullActivator.class` | Custom activator |
| `renderer` | `Class<? extends OptionRenderer>` | `NullOptionRenderer.class` | Custom renderer |
| `parser` | `Class<? extends OptionParser>` | `AeshOptionParser.class` | Custom parser |
| `aliases` | `String[]` | `{}` | Alternative long names for this option |
| `helpGroup` | `String` | `""` | Group heading for this option in help output (see [Options - Help Grouping](../options#help-grouping)) |
| `exclusiveWith` | `String[]` | `{}` | Names of mutually exclusive options (see [Options - Mutually Exclusive Options](../options#mutually-exclusive-options)) |
| `visibility` | `OptionVisibility` | `BRIEF` | Controls help and completion visibility (see [Options - Visibility Levels](../options#visibility-levels)) |
| `order` | `int` | `Integer.MAX_VALUE` | Explicit help-order position (see [Options - Help Option Ordering](../options#help-option-ordering)) |

## Basic Example

```java
@CommandDefinition(name = "build")
public class BuildCommand implements Command<CommandInvocation> {

    @OptionList(name = "modules", description = "Modules to build")
    private List<String> modules;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Building: " + String.join(", ", modules));
        return CommandResult.SUCCESS;
    }
}
```

Usage: `build --modules module1,module2,module3`
Or with space: `build --modules module1,module2,module3`

## Custom Separator

Use `valueSeparator` to change the delimiter:

```java
@OptionList(name = "ports", valueSeparator = ':', description = "Port list")
private List<Integer> ports;
```

Usage: `build --ports 8080:8081:8082`

## Value Attachment Syntaxes

`@OptionList` supports multiple ways to attach values, matching POSIX/GNU conventions:

```bash
# Space-separated (all shells)
-R -Xmx4G -R -Xms4G

# Equals-attached (short and long names)
-R=-Xmx4G
--runtime-option=-Xmx4G

# Bare-attached (short names only, since 3.16)
-R-Xmx4G -R-Xms4G
```

The bare-attached syntax (`-R-Xmx4G`) is common for JVM-style options where the value starts with a dash. This works for single-character short names without requiring `=` or a space:

```java
@OptionList(shortName = 'R', name = "runtime-option")
private List<String> javaRuntimeOptions;
```

```bash
# All equivalent:
myapp -R-Xmx4G -R-Xms4G server.jar
myapp -R=-Xmx4G -R=-Xms4G server.jar
myapp -R -Xmx4G -R -Xms4G server.jar
```

When using a custom `valueSeparator` (e.g., comma), the separator is applied within the attached value:

```bash
# -o with valueSeparator=','
myapp -ofoo,bar,baz    # three values: foo, bar, baz
myapp -o=foo,bar       # two values: foo, bar
```

## With Converter

```java
@OptionList(
    name = "files", 
    converter = PathConverter.class,
    description = "List of files"
)
private List<Path> files;

public static class PathConverter implements Converter<Path> {
    @Override
    public Path convert(String input) {
        return Paths.get(input);
    }
}
```

Usage: `process --files /path/file1.txt,/path/file2.txt`

## Default Values

```java
@OptionList(
    defaultValue = {"core", "utils", "api"},
    description = "Modules"
)
private List<String> modules;
```

## Set Collection

Use `Set` instead of `List` to eliminate duplicates:

```java
@OptionList(description = "Unique tags")
private Set<String> tags;
```

## Aliases

Provide alternative long names with `aliases`:

```java
@OptionList(name = "items", aliases = {"item"}, description = "Items to process")
private List<String> items;
```

Both `--items a,b,c` and `--item a,b,c` are accepted. See [Options - Option Aliases](../options#option-aliases) for more details.

## Custom Types

```java
@OptionList(converter = VersionConverter.class, description = "Supported versions")
private List<Version> versions;
```
