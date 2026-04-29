---
date: '2026-04-24T10:00:00+02:00'
draft: false
title: 'Annotation Processor'
weight: 18
---

The `aesh-processor` module provides a compile-time annotation processor that generates command metadata at build time, eliminating runtime reflection for annotation scanning, object instantiation, and field injection.

## Why Use It?

By default, Aesh uses runtime reflection to scan `@CommandDefinition`, `@Option`, `@Argument` and other annotations every time a command is registered. The annotation processor shifts this work to compile time:

- **3-4x faster startup** -- Benchmarks show the generated path is 3.4-3.8x faster than the reflection path for command registration
- **No runtime reflection** -- Annotation scanning, instance creation, and field get/set are all replaced by generated code
- **Lower memory usage** -- No reflection metadata retained in memory
- **GraalVM native-image friendly** -- Generated code uses direct `new` calls and cached field accessors instead of reflection, reducing the need for reflection configuration
- **Compile-time validation** -- Catch annotation errors at build time, not at runtime
- **Zero behavior change** -- Existing commands are unaffected; the processor is fully optional

### Performance

Startup benchmarks (100 commands, 3000 measured iterations after warmup):

| Benchmark | Generated | Reflection | Speedup |
|---|---|---|---|
| Flat commands (4 options each) | 40 us | 135 us | **3.4x** |
| Mixin commands | 50 us | 182 us | **3.6x** |
| Group commands (parent + 2 children) | 26 us | 93 us | **3.6x** |
| Nested groups (3-level hierarchy) | 13 us | 50 us | **3.8x** |

The speedup comes from eliminating annotation scanning, reflective field lookup, and exception-based class hierarchy walking during command registration.

## Installation

Add `aesh-processor` as a build-time dependency. The Java compiler will automatically discover and run the processor during compilation.

### Maven

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>aesh-processor</artifactId>
  <version>${aesh.version}</version>
  <scope>provided</scope>
</dependency>
```

If your project uses `annotationProcessorPaths` in the `maven-compiler-plugin` configuration (common in multi-module builds or when other annotation processors are present), you must also add the processor there:

```xml
<plugin>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <annotationProcessorPaths>
      <path>
        <groupId>org.aesh</groupId>
        <artifactId>aesh-processor</artifactId>
        <version>${aesh.version}</version>
      </path>
    </annotationProcessorPaths>
  </configuration>
</plugin>
```

When `annotationProcessorPaths` is configured, Maven uses only those paths for processor discovery and ignores the compile classpath. If you use this element for another processor (e.g. Lombok, MapStruct), you must add `aesh-processor` to the list or it will not run.

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

### What Gets Generated

For each option, argument, and mixin field, the generated code provides:

| Reflection operation | Generated replacement |
|---|---|
| Annotation scanning (`getAnnotation`, `getDeclaredFields`) | Compile-time literals |
| Instance creation (`ReflectionUtil.newInstance()`) | Direct `new MyCommand()` |
| Validator, converter, completer, activator creation | Direct `new X()` calls |
| Field set (`field.set(instance, value)`) | Cached `Field` + `fieldSetter` accessor |
| Field get (`field.get(instance)`) | Cached `Field` + `fieldGetter` accessor |
| Field reset (clear between parses) | Cached `Field` + `fieldResetter` accessor |
| `@ParentCommand` injection (field scanning + set) | Direct `parentCommandInjector` |
| Mixin field resolution (`getField` + `setAccessible`) | Cached mixin `Field` + two-step accessor |

The generated `Field` constants are resolved once in a `static {}` initializer and reused for every parse cycle, avoiding repeated class hierarchy walks.

## Generated Code

For a command like:

```java
@CommandDefinition(name = "build", description = "Run build")
public class BuildCommand implements Command<CommandInvocation> {
    @Option(shortName = 'v', description = "Verbose", hasValue = false)
    private boolean verbose;

    @Option(name = "output", required = true, description = "Output path")
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

    // Field references resolved once at class load
    private static final java.lang.reflect.Field FIELD_verbose;
    private static final java.lang.reflect.Field FIELD_outputFile;
    private static final java.lang.reflect.Field FIELD_source;

    static {
        try {
            FIELD_verbose = BuildCommand.class.getDeclaredField("verbose");
            FIELD_verbose.setAccessible(true);
            FIELD_outputFile = BuildCommand.class.getDeclaredField("outputFile");
            FIELD_outputFile.setAccessible(true);
            FIELD_source = BuildCommand.class.getDeclaredField("source");
            FIELD_source.setAccessible(true);
        } catch (NoSuchFieldException e) {
            throw new RuntimeException(e);
        }
    }

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

    public String commandName() { return "build"; }

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
                        .fieldSetter((inst, val) -> {
                            try { FIELD_verbose.set(inst, val); }
                            catch (IllegalAccessException e) { throw new RuntimeException(e); }
                        })
                        .fieldResetter(inst -> {
                            try { FIELD_verbose.set(inst, false); }
                            catch (IllegalAccessException e) { throw new RuntimeException(e); }
                        })
                        .fieldGetter(inst -> {
                            try { return FIELD_verbose.get(inst); }
                            catch (IllegalAccessException e) { throw new RuntimeException(e); }
                        })
                        .build());

        // ... output and source options with the same pattern

        return processedCommand;
    }
}
```

The generated code uses the same `ProcessedCommandBuilder` and `ProcessedOptionBuilder` APIs that the reflection path uses, ensuring identical behavior. The key difference is that field access uses cached `Field` constants with pre-built accessor functions, rather than scanning annotations and resolving fields at runtime.

## Compile-Time Validation

The processor validates commands at compile time and reports errors as compiler errors:

- Command class must not be abstract
- Command class must implement `Command`
- Command class must have an accessible no-arg constructor
- `@OptionList` and `@Arguments` fields must be `Collection` subtypes
- `@OptionGroup` fields must be `Map` subtypes

These checks catch errors that would otherwise only surface at runtime.

## Feature Support

The processor supports all Aesh annotation features:

### Group Commands

Group commands with `@GroupCommandDefinition` are fully supported. The processor generates metadata for the parent command and records its subcommand classes. At runtime, each subcommand is resolved through its own provider (if available) or via the reflection fallback.

```java
@GroupCommandDefinition(name = "remote", description = "Manage remotes",
        groupCommands = {RemoteAddCommand.class, RemoteRemoveCommand.class})
public class RemoteCommand implements Command<CommandInvocation> {
    // ...
}
```

### Class Hierarchies

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

### Mixins

[Mixins](mixins) are fully supported. The generated code resolves mixin fields at compile time and generates two-step accessors that navigate through the mixin instance:

```java
@CommandDefinition(name = "build", description = "Build project")
public class BuildCommand implements Command<CommandInvocation> {
    @Mixin
    LoggingMixin logging;  // auto-created if null

    @Option(description = "Target")
    private String target;
}
```

The generated metadata caches both the mixin field (`FIELD_logging`) and each option's field within the mixin (e.g., `FIELD_logging_verbose`). The generated `fieldSetter` navigates through the mixin instance in a single call, avoiding the per-option mixin field resolution that the reflection path performs.

### @ParentCommand

The processor generates a `parentCommandInjector` for commands with `@ParentCommand` fields. This eliminates the runtime field scanning that would otherwise search all declared fields for the annotation:

```java
@CommandDefinition(name = "add", description = "Add remote")
public class RemoteAddCommand implements Command<CommandInvocation> {
    @ParentCommand
    RemoteCommand parent;  // injected via generated accessor, no field scanning
}
```

### All Option Features

Custom validators, completers, converters, activators, renderers, result handlers, option parsers, default value providers, optional-value options, negatable options, stop-at-positional, inherited options, exclusive options, option visibility levels, and help section providers are all supported.

## Compatibility

- **Java 8+** -- The processor targets source version 8
- **No code changes required** -- Drop in the dependency and the processor runs automatically
- **Fully backward compatible** -- Removing the dependency reverts to the reflection path with no behavior change
- **Incremental** -- You can use the processor for some commands and not others; commands without a generated provider fall back to reflection
