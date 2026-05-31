---
date: '2026-04-24T10:00:00+02:00'
draft: false
title: 'Annotation Processor'
weight: 18
---

The `aesh-processor` module provides a compile-time annotation processor that generates command metadata at build time, eliminating runtime reflection for annotation scanning, object instantiation, and field injection.

## Why Use It?

By default, Aesh uses runtime reflection to scan `@CommandDefinition`, `@Option`, `@Argument` and other annotations every time a command is registered. The annotation processor shifts this work to compile time:

- **5-8x faster startup** -- Benchmarks show the generated path is 5-8x faster than the reflection path for command registration
- **No runtime reflection** -- Annotation scanning, instance creation, and field get/set are all replaced by generated code
- **Lower memory usage** -- No reflection metadata retained in memory; converters and completers are shared across options
- **GraalVM native-image friendly** -- Generated code uses direct `new` calls and cached field accessors instead of reflection, reducing the need for reflection configuration
- **Compile-time validation** -- Catch annotation errors at build time, not at runtime
- **Zero behavior change** -- Existing commands are unaffected; the processor is fully optional

### Performance

Startup benchmarks (100 commands, 3000 measured iterations after warmup):

| Benchmark | Generated | Reflection | Speedup |
|---|---|---|---|
| Flat commands (4 options each) | 22 us | 153 us | **6.8x** |
| Mixin commands | 26 us | 208 us | **8.0x** |
| Group commands (parent + 2 children) | 15 us | 102 us | **7.0x** |
| Nested groups (3-level hierarchy) | 10 us | 56 us | **5.8x** |
| Option variety (OptionList, OptionGroup, help, version) | 20 us | 135 us | **6.8x** |

The speedup comes from bypassing the `ProcessedOptionBuilder` entirely (using `ProcessedOption.createDirect()`), skipping compile-time-verified uniqueness checks (`addOptionDirect()`), and sharing converter and completer instances across options via static constants.

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

1. At compile time, the processor scans classes annotated with `@CommandDefinition` (or the deprecated `@GroupCommandDefinition`) and generates a `_AeshMetadata` class for each command
2. A single `_AeshMetadataRegistry` class is generated with a `switch` statement that maps command class names to their metadata providers
3. The registry implements `MetadataRegistry` and is registered via `META-INF/services` (ServiceLoader)
4. At runtime, when Aesh registers a command, ServiceLoader discovers the single registry class (not all 65+ individual providers). Only the requested command's metadata class is loaded and instantiated via the `switch` -- other commands remain untouched
5. If no provider is found, Aesh falls back to the existing reflection-based path

```
Compile Time                          Runtime
┌─────────────────────┐
│  @CommandDefinition │
│  MyCommand.java     │
│  OtherCommand.java  │
└─────────┬───────────┘
          │ annotation processor
          ▼
┌─────────────────────────┐     ┌──────────────────────────────┐
│ MyCommand_AeshMetadata  │     │ ServiceLoader discovers      │
│ OtherCmd_AeshMetadata   │     │ _AeshMetadataRegistry (1 cls)│
│ _AeshMetadataRegistry   │ ──▶ │                              │
│  switch("MyCommand")    │     │ switch hit ──▶ new MyCommand_AeshMetadata()
│    → new MyCmd_Meta()   │     │ switch miss ──▶ Reflection fallback
└─────────────────────────┘     └──────────────────────────────┘
```

This architecture means a project with 65 commands (like jbang) loads only 3 classes at startup (the registry + the invoked command's metadata + the command itself) instead of 130 classes (65 providers + 65 commands).

### What Gets Generated

For each option, argument, and mixin field, the generated code provides:

| Reflection operation | Generated replacement |
|---|---|
| Annotation scanning (`getAnnotation`, `getDeclaredFields`) | Compile-time literals |
| Instance creation (`ReflectionUtil.newInstance()`) | Direct `new MyCommand()` |
| Validator, converter, completer, activator creation | Direct `new X()` calls |
| Field set (`field.set(instance, value)`) | Direct assignment or cached `Field` accessor |
| Field get (`field.get(instance)`) | Direct read or cached `Field` accessor |
| Field reset (clear between parses) | Direct assignment or cached `Field` accessor |
| `@ParentCommand` injection (field scanning + set) | Direct `parentCommandInjector` |
| Mixin field resolution (`getField` + `setAccessible`) | Direct mixin field access or cached `Field` |

### Field Visibility

{{< callout type="info" >}}
**Recommendation:** Use `public` or package-private fields for `@Option`, `@Argument`, and `@Mixin` fields when using the annotation processor. This gives **zero-reflection** performance.
{{< /callout >}}

- **Public / package-private fields** -- The generated code uses direct field access (`((MyCommand) inst).myField = value`). No `java.lang.reflect.Field`, no `getDeclaredField()`, no `setAccessible()`. This is the fastest possible path and requires no GraalVM native-image reflection configuration.

- **Private fields** -- The generated code resolves `Field` constants once in a `static {}` initializer and caches them for reuse. This avoids repeated class hierarchy walks but still uses `getDeclaredField()` + `setAccessible()` at class load time, and requires reflection entries in native-image configuration (which the processor generates automatically -- see [GraalVM Native Image](#graalvm-native-image) below).

## Generated Code

For a command with public fields (recommended):

```java
@CommandDefinition(name = "build", description = "Run build")
public class BuildCommand implements Command<CommandInvocation> {
    @Option(shortName = 'v', description = "Verbose", hasValue = false)
    boolean verbose;

    @Option(name = "output", required = true, description = "Output path")
    String outputFile;

    @Argument(description = "Source")
    String source;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        return CommandResult.SUCCESS;
    }
}
```

The processor generates `BuildCommand_AeshMetadata` with **zero reflection** -- direct field access via a switch-based `Accessor`:

```java
public final class BuildCommand_AeshMetadata
        implements CommandMetadataProvider<BuildCommand> {

    // No Field constants needed -- direct access to package-private fields

    // Shared converter and completer constants
    private static final Converter CONVERTER_Boolean = CLConverterManager.getInstance()
            .getConverter(Boolean.class);
    private static final OptionCompleter BOOLEAN_COMPLETER = new BooleanOptionCompleter();

    public Class<BuildCommand> commandType() { return BuildCommand.class; }
    public BuildCommand newInstance() { return new BuildCommand(); }
    public boolean isGroupCommand() { return false; }
    public Class<? extends Command>[] groupCommandClasses() { return new Class[0]; }
    public String commandName() { return "build"; }

    public ProcessedCommand buildProcessedCommand(BuildCommand instance) {
        // Options use ProcessedOption.createDirect() with an Accessor
        // for reflection-free field get/set via switch dispatch
        // ...
    }

    // Switch-based accessor -- no reflection, no lambdas
    static final class Accessor implements FieldAccessor, BiConsumer<Object, Object> {
        private final int idx;
        Accessor(int idx) { this.idx = idx; }

        public void set(Object inst, Object val) {
            switch (idx) {
                case 0: ((BuildCommand) inst).verbose = val != null && (Boolean) val; break;
                case 1: ((BuildCommand) inst).outputFile = (String) val; break;
                case 2: ((BuildCommand) inst).source = (String) val; break;
            }
        }
        public Object get(Object inst) {
            switch (idx) {
                case 0: return ((BuildCommand) inst).verbose;
                case 1: return ((BuildCommand) inst).outputFile;
                case 2: return ((BuildCommand) inst).source;
                default: return null;
            }
        }
        // ...
    }
}
```

For commands with **private** fields, the generated code uses cached `Field` constants with `getDeclaredField()` + `setAccessible()` in a static initializer, and the `Accessor` switch delegates to those fields. See [Field Visibility](#field-visibility) above.

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

Group commands are fully supported using `@CommandDefinition` with `groupCommands`:

```java
@CommandDefinition(name = "remote", description = "Manage remotes",
        groupCommands = {RemoteAddCommand.class, RemoteRemoveCommand.class})
public class RemoteCommand implements Command<CommandInvocation> {
    // ...
}
```

The processor generates metadata for the parent command and records its subcommand classes. At runtime, each subcommand is resolved lazily through its own provider (if available) or via the reflection fallback.

{{< callout type="warning" >}}
`@GroupCommandDefinition` is deprecated. Use `@CommandDefinition(groupCommands = {...})` instead. The processor supports both, but new code should use the unified annotation.
{{< /callout >}}

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

## GraalVM Native Image

The processor automatically generates GraalVM native-image configuration files under `META-INF/native-image/org.aesh/<project>/`:

- **`resource-config.json`** -- Always generated. Ensures the `META-INF/services/org.aesh.command.metadata.MetadataRegistry` ServiceLoader descriptor is included in the native image.

- **`reflect-config.json`** -- Generated only when commands have **private** annotated fields. Contains entries for each class with private `@Option`, `@Argument`, `@Mixin`, or `@ParentCommand` fields, enabling `getDeclaredField()` + `setAccessible()` in the generated code.

Commands with only public/package-private fields produce **no reflection entries** -- the generated code accesses them directly.

### Processor Options

Configure via `-A` compiler flags:

| Option | Default | Description |
|---|---|---|
| `aeshNativeImageProject` | `aesh-generated` | Subdirectory name under `META-INF/native-image/org.aesh/` |
| `aeshNativeImageDisable` | `false` | Set to `true` to skip native-image config generation |

Example Maven configuration:

```xml
<plugin>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <compilerArgs>
      <arg>-AaeshNativeImageProject=my-app</arg>
    </compilerArgs>
  </configuration>
</plugin>
```

## Compatibility

- **Java 8+** -- The processor targets source version 8
- **No code changes required** -- Drop in the dependency and the processor runs automatically
- **Fully backward compatible** -- Removing the dependency reverts to the reflection path with no behavior change
- **Incremental** -- You can use the processor for some commands and not others; commands without a generated provider fall back to reflection
- **Multi-module** -- Each module generates its own `_AeshMetadataRegistry`. ServiceLoader merges registries across JARs automatically
