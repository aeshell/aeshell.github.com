---
date: '2026-03-12T12:00:00+01:00'
draft: false
title: 'Annotation Processor'
weight: 18
---

The `aesh-processor` module provides a compile-time annotation processor that generates command metadata at build time, eliminating runtime reflection for annotation scanning and object instantiation.

## Why Use It?

By default, Aesh uses runtime reflection to scan `@CommandDefinition`, `@Option`, `@Argument` and other annotations every time a command is registered. The annotation processor shifts this work to compile time:

- **Faster startup** -- No annotation scanning or reflective instantiation at runtime
- **Lower memory usage** -- No reflection metadata retained in memory
- **GraalVM native-image friendly** -- Generated code uses direct `new` calls instead of reflection, reducing the need for reflection configuration
- **Zero behavior change** -- Existing users are unaffected; the processor is fully optional

## Installation

Add `aesh-processor` as a `provided` dependency. The Java compiler will automatically discover and run the processor during compilation.

### Maven

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>aesh-processor</artifactId>
  <version>${aesh.version}</version>
  <scope>provided</scope>
</dependency>
```

### Gradle

```groovy
dependencies {
    annotationProcessor 'org.aesh:aesh-processor:${aeshVersion}'
    compileOnly 'org.aesh:aesh-processor:${aeshVersion}'
}
```

## How It Works

The processor follows a **dual-mode** approach:

1. At compile time, the processor scans classes annotated with `@CommandDefinition` or `@GroupCommandDefinition` and generates a `_AeshMetadata` class for each command
2. The generated classes implement `CommandMetadataProvider` and are registered via `META-INF/services` (ServiceLoader)
3. At runtime, when Aesh registers a command, it first checks if a generated provider exists for that command class
4. If a provider is found, it uses the generated metadata (no reflection). If not, it falls back to the existing reflection-based path

```
Compile Time                          Runtime
┌─────────────────────┐
│  @CommandDefinition │
│  MyCommand.java     │
└─────────┬───────────┘
          │ annotation processor
          ▼
┌─────────────────────────┐     ┌───────────────────────┐
│ MyCommand_AeshMetadata  │ ──▶ │ ServiceLoader lookup  │
│  (generated source)     │     │ Provider found? ──Yes──▶ Use generated metadata
└─────────────────────────┘     │                ──No───▶ Reflection fallback
                                └───────────────────────┘
```

### What Gets Eliminated

| Reflection operation | With processor |
|---|---|
| Annotation scanning (`getAnnotation`, `getDeclaredFields`) | Replaced by compile-time literals |
| Instance creation for validators, converters, completers, activators, renderers, result handlers | Replaced by direct `new X()` calls |
| Command instantiation via `ReflectionUtil.newInstance()` | Replaced by `new MyCommand()` |
| Field injection (`field.set`) | Still uses reflection (fields are private) |

## Generated Code

For a command like:

```java
@CommandDefinition(name = "build", description = "Run build")
public class BuildCommand implements Command<CommandInvocation> {
    @Option(shortName = 'v', description = "Verbose")
    private boolean verbose;

    @Option(name = "output", required = true)
    private String outputFile;

    @Argument(description = "Source")
    private String source;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        return CommandResult.SUCCESS;
    }
}
```

The processor generates `BuildCommand_AeshMetadata` in the same package:

```java
public final class BuildCommand_AeshMetadata
        implements CommandMetadataProvider<BuildCommand> {

    public Class<BuildCommand> commandType() {
        return BuildCommand.class;
    }

    public BuildCommand newInstance() {
        return new BuildCommand();  // no reflection
    }

    public boolean isGroupCommand() { return false; }

    public Class<? extends Command>[] groupCommandClasses() {
        return new Class[0];
    }

    public ProcessedCommand buildProcessedCommand(BuildCommand instance)
            throws CommandLineParserException {
        ProcessedCommand processedCommand = ProcessedCommandBuilder.builder()
                .name("build")
                .description("Run build")
                .command(instance)
                .create();

        processedCommand.addOption(
                ProcessedOptionBuilder.builder()
                        .shortName('v')
                        .name("verbose")
                        .description("Verbose")
                        .type(boolean.class)
                        .fieldName("verbose")
                        .optionType(OptionType.BOOLEAN)
                        .completer(new BooleanOptionCompleter())
                        // ... other literal values
                        .build());

        // ... more options and argument

        return processedCommand;
    }
}
```

The generated code uses the same `ProcessedCommandBuilder` and `ProcessedOptionBuilder` APIs that the reflection path uses, ensuring identical behavior.

## Compile-Time Validation

The processor validates commands at compile time and reports errors as compiler errors:

- Command class must not be abstract
- Command class must implement `Command`
- Command class must have an accessible no-arg constructor
- `@OptionList` and `@Arguments` fields must be `Collection` subtypes
- `@OptionGroup` fields must be `Map` subtypes

These checks catch errors that would otherwise only surface at runtime.

## Group Commands

Group commands with `@GroupCommandDefinition` are fully supported. The processor generates metadata for the parent command and records its subcommand classes. At runtime, each subcommand is resolved through its own provider (if available) or via the reflection fallback.

```java
@GroupCommandDefinition(name = "remote", description = "Manage remotes",
        groupCommands = {RemoteAddCommand.class, RemoteRemoveCommand.class})
public class RemoteCommand implements Command<CommandInvocation> {
    // ...
}
```

## Class Hierarchies

The processor walks the full class hierarchy, collecting annotated fields from superclasses. If your commands extend a base class with shared options, those options are included in the generated metadata.

```java
public abstract class BaseCommand implements Command<CommandInvocation> {
    @Option(description = "Enable debug mode", hasValue = false)
    private boolean debug;
}

@CommandDefinition(name = "deploy", description = "Deploy application")
public class DeployCommand extends BaseCommand {
    @Option(description = "Target environment")
    private String environment;
    // Generated metadata includes both 'debug' and 'environment'
}
```

## Compatibility

- **Java 8+** -- The processor targets source version 8
- **No code changes required** -- Drop in the dependency and the processor runs automatically
- **Fully backward compatible** -- Removing the dependency reverts to the reflection path with no behavior change
- **Works with all Aesh features** -- Custom validators, completers, converters, activators, renderers, result handlers, option parsers, default value providers, optional-value options, negatable options, stop-at-positional, and inherited options are all supported
