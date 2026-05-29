---
date: '2026-04-29T12:00:00+02:00'
draft: false
title: 'Migrating from Picocli'
weight: 19
---

This guide helps you migrate CLI applications from [picocli](https://picocli.info/) to Aesh. The two frameworks use similar annotation-based models, so most migrations are straightforward renaming.

## Why Migrate?

**Startup performance.** Aesh registers commands 67-655x faster than picocli in benchmarks, depending on command complexity. With the optional [annotation processor](../annotation-processor), Aesh eliminates all runtime reflection and is a further 3-4x faster than its own reflection path.

| Benchmark (100 commands) | Aesh (generated) | Picocli (reflection) | Speedup |
|---|---|---|---|
| Flat commands (4 options each) | 40 us | 2,646 us | **67x** |
| Group commands (parent + 2 children) | 26 us | 7,242 us | **282x** |
| Nested groups (3-level hierarchy) | 13 us | 8,776 us | **655x** |

Aesh also parses command lines 8-21x faster than picocli once commands are registered.

**GraalVM native-image.** The annotation processor generates direct `new` calls and cached field accessors, reducing the need for reflection configuration in native images.

**Smaller footprint.** Aesh has no transitive dependencies beyond aesh-readline. Picocli is a single jar but is significantly larger.

## Annotation Mapping

### Commands

| Picocli | Aesh |
|---------|------|
| `@Command(name = "deploy", description = "...")` | `@CommandDefinition(name = "deploy", description = "...")` |
| `@Command(subcommands = {Sub.class})` | `@GroupCommandDefinition(name = "grp", groupCommands = {Sub.class})` |
| `@Command(mixinStandardHelpOptions = true)` | `@CommandDefinition(generateHelp = true)` |
| `@Command(version = "1.0")` | `@CommandDefinition(version = "1.0")` |
| `implements Runnable` or `Callable<Integer>` | `implements Command<CommandInvocation>` |

In picocli, `@Command` handles both simple and group commands. In Aesh, group commands use the separate `@GroupCommandDefinition` annotation with a `groupCommands` attribute listing subcommand classes.

### Options

| Picocli | Aesh |
|---------|------|
| `@Option(names = {"-v", "--verbose"})` | `@Option(shortName = 'v', name = "verbose")` |
| `@Option(names = "-v", arity = "0")` | `@Option(shortName = 'v', hasValue = false)` |
| `@Option(names = "--count", defaultValue = "10")` | `@Option(name = "count", defaultValue = "10")` |
| `@Option(names = "--out", required = true)` | `@Option(name = "out", required = true)` |
| `@Option(names = "--items", split = ",") List<String>` | `@OptionList(name = "items", valueSeparator = ',')` |
| `@Option(names = "-D") Map<String,String>` | `@OptionGroup(shortName = 'D')` |
| `@Option mapFallbackValue = ""` | `@OptionGroup(defaultValue = "")` |

Key differences:
- Aesh `shortName` is a `char`, not a string. The long name defaults to the field name or is set via `name`.
- Boolean flags require `hasValue = false` in Aesh (picocli infers this from the field type or `arity = "0"`).
- List options use the dedicated `@OptionList` annotation instead of putting `split` on `@Option`.
- Map/property options use `@OptionGroup` instead of type inference on `Map` fields.

### Arguments

| Picocli | Aesh |
|---------|------|
| `@Parameters(index = "0")` | `@Argument(index = "0")` |
| `@Parameters(index = "1..*") List<String>` | `@Arguments(index = "1..*")` |
| `@Parameters(description = "...")` | `@Argument(description = "...")` |
| `@Parameters(paramLabel = "FILE")` | `@Argument(paramLabel = "FILE")` |
| `@Parameters(arity = "1..*")` | `@Arguments(arity = "1..*")` |
| `@Parameters(arity = "2")` | `@Arguments(arity = "2")` |

Aesh uses `@Argument` for singular positional values and `@Arguments` for multiple values. Both support explicit `index` ranges, so picocli `@Parameters(index = ...)` maps directly. The `paramLabel` attribute customizes the display name in help synopsis, and `arity` constrains how many values are accepted (e.g., `"2"` for exactly two, `"1..*"` for one-or-more).

### Other Annotations

| Picocli | Aesh |
|---------|------|
| `@Mixin` | `@Mixin` |
| `@ParentCommand` | `@ParentCommand` |

These work the same way in both frameworks.

### Execution

| Picocli | Aesh |
|---------|------|
| `new CommandLine(cmd).execute(args)` | `AeshRuntimeRunner.builder().command(Cmd.class).args(args).execute()` |
| `new CommandLine(cmd)` + interactive loop | `AeshConsoleRunner.builder().command(Cmd.class).prompt("$ ").start()` |

## Before and After

### Picocli

```java
import picocli.CommandLine;
import picocli.CommandLine.Command;
import picocli.CommandLine.Option;
import picocli.CommandLine.Parameters;

@Command(name = "deploy", description = "Deploy an application",
         mixinStandardHelpOptions = true, version = "1.0")
public class DeployCommand implements Callable<Integer> {

    @Option(names = {"-e", "--environment"}, defaultValue = "production",
            description = "Target environment")
    private String environment;

    @Option(names = {"-f", "--force"}, description = "Force deployment")
    private boolean force;

    @Option(names = {"-t", "--tags"}, split = ",",
            description = "Deployment tags")
    private List<String> tags;

    @Option(names = "-D", description = "Properties")
    private Map<String, String> properties;

    @Parameters(index = "0", description = "Application name")
    private String application;

    @Override
    public Integer call() {
        System.out.println("Deploying " + application + " to " + environment);
        return 0;
    }

    public static void main(String[] args) {
        int exitCode = new CommandLine(new DeployCommand()).execute(args);
        System.exit(exitCode);
    }
}
```

### Aesh

```java
import org.aesh.command.*;
import org.aesh.command.invocation.CommandInvocation;
import org.aesh.command.option.*;
import org.aesh.AeshRuntimeRunner;

import java.util.List;
import java.util.Map;

@CommandDefinition(name = "deploy", description = "Deploy an application",
                   generateHelp = true, version = "1.0")
public class DeployCommand implements Command<CommandInvocation> {

    @Option(shortName = 'e', name = "environment", defaultValue = "production",
            description = "Target environment")
    private String environment;

    @Option(shortName = 'f', name = "force", hasValue = false,
            description = "Force deployment")
    private boolean force;

    @OptionList(shortName = 't', name = "tags", valueSeparator = ',',
                description = "Deployment tags")
    private List<String> tags;

    @OptionGroup(shortName = 'D', description = "Properties")
    private Map<String, String> properties;

    @Argument(description = "Application name", required = true)
    private String application;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Deploying " + application + " to " + environment);
        return CommandResult.SUCCESS;
    }

    public static void main(String[] args) {
        AeshRuntimeRunner.builder()
                .command(DeployCommand.class)
                .args(args)
                .execute();
    }
}
```

### What Changed

1. `@Command` became `@CommandDefinition` with `generateHelp = true`
2. `implements Callable<Integer>` became `implements Command<CommandInvocation>`
3. `@Option(names = {"-f", "--force"})` became `@Option(shortName = 'f', name = "force", hasValue = false)` -- boolean flags need `hasValue = false`
4. `@Option(split = ",") List<String>` became `@OptionList(valueSeparator = ',')`
5. `Map<String, String>` with `@Option` became `@OptionGroup`
6. `@Parameters` became `@Argument`
7. `System.out.println` became `invocation.println`
8. Return `CommandResult.SUCCESS` instead of an exit code integer
9. `CommandLine.execute(args)` became `AeshRuntimeRunner.builder()...execute()`

## Key Differences

### Output

Picocli commands use `System.out` directly. Aesh commands use `CommandInvocation.println()`, which routes output through the terminal connection. This enables the same command to work with local terminals, SSH, telnet, and WebSocket connections.

### Exit Codes

Picocli commands return an `int` exit code from `Callable.call()`. Aesh commands return `CommandResult.SUCCESS`, `CommandResult.FAILURE`, or `CommandResult.valueOf(code)`. Parse errors automatically return exit code 2 (POSIX convention).

### Group Commands

Picocli uses `subcommands = {...}` on `@Command`. Aesh uses a separate `@GroupCommandDefinition` annotation:

```java
// Picocli
@Command(name = "remote", subcommands = {AddCommand.class, RemoveCommand.class})
public class RemoteCommand implements Runnable { ... }

// Aesh
@GroupCommandDefinition(name = "remote",
        groupCommands = {AddCommand.class, RemoveCommand.class})
public class RemoteCommand implements Command<CommandInvocation> { ... }
```

### Inherited Options

Picocli supports inherited options via `@Option(scope = INHERIT)`. Aesh uses `@Option(inherited = true)` on the parent command. Inherited options can appear before or after the subcommand name on the command line:

```java
@GroupCommandDefinition(name = "app", groupCommands = {RunCommand.class})
public class AppCommand implements Command<CommandInvocation> {
    @Option(name = "verbose", hasValue = false, inherited = true)
    boolean verbose;
}
```

Both `app --verbose run` and `app run --verbose` set `verbose = true` on the child command.

## Aesh-Only Features

These features have no picocli equivalent:

- **[Negatable booleans](../options#negatable-options)** -- `@Option(negatable = true)` generates `--no-verbose` automatically
- **[Exclusive options](../options#mutually-exclusive-options)** -- `@Option(exclusiveWith = "json")` enforces mutual exclusion at parse time
- **[Option visibility](../options#visibility-levels)** -- `OptionVisibility.BRIEF/FULL/HIDDEN` controls which options appear in `--help` and tab completion
- **[Interactive prompting](../options)** -- `@Option(askIfNotSet = true)` prompts the user for missing required values
- **[Ghost text suggestions](../ghost-text-suggestions)** -- Inline suggestions as the user types
- **[Selectors](../selectors)** -- Interactive list/checkbox selection for option values
- **[Sub-command mode](../sub-command-mode)** -- Enter a context where subcommands run without repeating the parent name

## Shell Completion Scripts

Aesh generates bash, zsh, and fish completion scripts automatically -- no extra code required. The standard `AeshRuntimeRunner` pattern automatically supports `--aesh-completion` and `--aesh-completion-install` flags. See [Completers -- Shell Completion Scripts](../completers#shell-completion-scripts) for details.

```bash
# Generate completion script (auto-detects shell)
$ myapp --aesh-completion

# Generate for a specific shell
$ myapp --aesh-completion bash

# Auto-detect, generate, and install
$ myapp --aesh-completion-install
```

Picocli's `AutoComplete` only supports bash. Aesh supports bash, zsh, and fish, with both static and dynamic (callback-based) completion scripts.

## Annotation Processor

For maximum startup performance, add the [annotation processor](../annotation-processor) to eliminate all runtime reflection:

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>aesh-processor</artifactId>
  <version>${aesh.version}</version>
  <scope>provided</scope>
</dependency>
```

No code changes needed -- the processor runs at compile time and the runtime automatically uses the generated metadata when available. See [Annotation Processor](../annotation-processor) for setup details.
