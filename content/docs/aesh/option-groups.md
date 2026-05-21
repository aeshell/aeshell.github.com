---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Option Groups'
weight: 6
---

The `@OptionGroup` annotation defines key=value pair options collected into a `Map`. This is useful for options like `-Dkey=value` where the same flag can be repeated with different keys.

## Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `name` | `String` | `""` | Option name (variable name if empty) |
| `shortName` | `char` | `'\u0000'` | Short name (e.g., `-D`) |
| `description` | `String` | `""` | Help description |
| `required` | `boolean` | `false` | Is option required? |
| `defaultValue` | `String[]` | `{}` | Default values |
| `askIfNotSet` | `boolean` | `false` | Prompt user if not set |
| `converter` | `Class<? extends Converter>` | `NullConverter.class` | Custom value converter |
| `completer` | `Class<? extends OptionCompleter>` | `NullOptionCompleter.class` | Custom completer |
| `validator` | `Class<? extends OptionValidator>` | `NullValidator.class` | Custom validator |
| `activator` | `Class<? extends OptionActivator>` | `NullActivator.class` | Custom activator |
| `renderer` | `Class<? extends OptionRenderer>` | `NullOptionRenderer.class` | Custom renderer |
| `parser` | `Class<? extends OptionParser>` | `AeshOptionParser.class` | Custom parser |
| `visibility` | `OptionVisibility` | `BRIEF` | Controls help and completion visibility (see [Options - Visibility Levels](../options#visibility-levels)) |
| `order` | `int` | `Integer.MAX_VALUE` | Explicit help-order position (see [Options - Help Option Ordering](../options#help-option-ordering)) |

## Basic Example

```java
@CommandDefinition(name = "compile", description = "Compile the project")
public class CompileCommand implements Command<CommandInvocation> {

    @OptionGroup(shortName = 'D', description = "Define system properties")
    private Map<String, String> properties;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (properties != null) {
            properties.forEach((key, value) ->
                invocation.println(key + " = " + value));
        }
        return CommandResult.SUCCESS;
    }
}
```

Usage: `compile -Denv=prod -Ddebug=true -Dlog.level=INFO`

Each `-D` occurrence adds an entry to the map. The key and value are split on the `=` character.

### Help Output

`@OptionGroup` renders with `=<key=value>` syntax in help to show the expected format:

```
Options:
  -D=<key=value>  Define system properties
```

Synopsis:
```
Usage: compile [-D<key>=<value>]
```

### Short-Name Only

When `@OptionGroup` has a `shortName` but no explicit `name`, only the short name is shown. No long name is auto-derived from the Java field name:

```java
@OptionGroup(shortName = 'D', description = "System properties")
private Map<String, String> properties;
// Renders as: -D=<key=value>  (NOT -D, --properties=<key=value>)
```

To also have a long name, set it explicitly: `@OptionGroup(shortName = 'D', name = "define", ...)`

## Field Type Requirements

The annotated field **must** implement `Map`. Both `Map` and `TreeMap` (or other `Map` implementations) are supported:

```java
// String keys and values
@OptionGroup(shortName = 'D')
private Map<String, String> define;

// String keys with typed values (uses built-in converters)
@OptionGroup(shortName = 'D', description = "Define properties")
private TreeMap<String, Integer> limits;
```

When using typed values, the value portion is converted using the built-in converter for that type, or a custom converter if specified.

## Real-World Examples

### Java-Style System Properties

```java
@CommandDefinition(name = "run", description = "Run application")
public class RunCommand implements Command<CommandInvocation> {

    @OptionGroup(shortName = 'D', description = "System properties")
    private Map<String, String> systemProperties;

    @Option(description = "Main class")
    private String mainClass;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (systemProperties != null) {
            systemProperties.forEach(System::setProperty);
        }
        // run application...
        return CommandResult.SUCCESS;
    }
}
```

Usage:
```bash
$ run --mainClass com.example.App -Dserver.port=8080 -Dspring.profiles.active=dev
```

### Build Configuration

```java
@CommandDefinition(name = "build", description = "Build project")
public class BuildCommand implements Command<CommandInvocation> {

    @OptionGroup(shortName = 'P', description = "Build parameters")
    private Map<String, String> params;

    @OptionGroup(shortName = 'E', description = "Environment overrides")
    private Map<String, String> env;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Access individual values from the map
        String version = params != null ? params.get("version") : "1.0";
        invocation.println("Building version " + version);
        return CommandResult.SUCCESS;
    }
}
```

Usage:
```bash
$ build -Pversion=2.0 -Ptarget=release -Ehome=/opt/app
```

## Keys Without Values

By default, `@OptionGroup` requires `key=value` format. Specifying just a key (e.g., `-Dfoo`) without `=` results in a parse error.

When a `defaultValue` is set, keys without `=` are accepted and the default value is used:

```java
@OptionGroup(shortName = 'D', description = "System properties",
             defaultValue = "")
private Map<String, String> properties;
```

Usage:
```bash
$ run -Dverbose -Dport=8080
```

This produces `{"verbose": "", "port": "8080"}`. The key `verbose` gets the default value (empty string), while `port` gets the explicit value `8080`.

This is useful for Java-style `-D` flags where `-Dflag` is equivalent to setting the property to an empty string:

```java
@OptionGroup(shortName = 'D', description = "System properties",
             defaultValue = "true")
private Map<String, String> properties;
```

With `defaultValue = "true"`, `-Dverbose` produces `{"verbose": "true"}`.

The same default value applies when `=` is present but the value is empty: `-Dkey=` also uses the default value.

## Differences from @OptionList

| | `@OptionGroup` | `@OptionList` |
|---|---|---|
| **Field type** | `Map<K, V>` | `List<T>` or `Set<T>` |
| **Input format** | `-Dkey=value` (repeated) | `--opt val1,val2,val3` |
| **Use case** | Key-value pairs | Simple value collections |
| **Repetition** | Same flag repeated with different keys | Single flag with separated values |
