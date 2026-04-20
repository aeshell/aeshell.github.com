---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Completers'
weight: 8
---

Completers provide tab-completion suggestions for options and arguments.

{{< callout type="info" >}}
Looking for inline ghost text suggestions instead of tab completion? See [Ghost Text Suggestions](../ghost-text-suggestions) for automatic command, subcommand, and option suggestions as you type.
{{< /callout >}}

## OptionCompleter Interface

```java
public interface OptionCompleter<T extends CompleterInvocation> {
    void complete(T completerInvocation);
}
```

## Built-in Completers

Æsh includes built-in completers:
- `BooleanOptionCompleter` - Completes boolean values
- `DefaultValueOptionCompleter` - Uses defined default values
- `FileOptionCompleter` - Completes file paths

## Custom Color Completer

```java
public class ColorCompleter implements OptionCompleter<CompleterInvocation> {

    private static final List<String> COLORS = Arrays.asList(
            "red", "green", "blue", "yellow", "orange", "purple",
            "cyan", "magenta", "white", "black", "gray", "pink"
    );

    @Override
    public void complete(CompleterInvocation invocation) {
        String input = invocation.getGivenCompleteValue();
        
        if (input == null || input.isEmpty()) {
            // No input yet, show all colors
            invocation.addAllCompleterValues(COLORS);
        } else {
            // Filter colors that start with the input
            String lowerInput = input.toLowerCase();
            for (String color : COLORS) {
                if (color.startsWith(lowerInput)) {
                    invocation.addCompleterValue(color);
                }
            }
        }
    }
}
```

Usage:

```java
@CommandDefinition(name = "theme", description = "Set application theme")
public class ThemeCommand implements Command<CommandInvocation> {

    @Option(
        shortName = 'b',
        name = "background",
        completer = ColorCompleter.class,
        description = "Background color"
    )
    private String backgroundColor;

    @Option(
        shortName = 'f',
        name = "foreground",
        completer = ColorCompleter.class,
        description = "Foreground/text color"
    )
    private String foregroundColor;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Setting theme: " + foregroundColor + " on " + backgroundColor);
        return CommandResult.SUCCESS;
    }
}
```

When users press Tab:
```
[myapp]$ theme --background <TAB>
red     green   blue    yellow  orange  purple
cyan    magenta white   black   gray    pink

[myapp]$ theme --background gr<TAB>
gray    green

[myapp]$ theme --background green --foreground wh<TAB>
white
```

## CompleterInvocation

Provides access to:
- `getGivenCompleteValue()` - Current partial input
- `getAllGivenValues()` - All values for this option
- `getCommand()` - The command instance
- `getAeshContext()` - Access to runtime context
- `addCompleterValue(String)` - Add a completion suggestion

## Dynamic Completion Based on Other Options

```java
public class DatabaseTableCompleter implements OptionCompleter<CompleterInvocation> {

    @Override
    public void complete(CompleterInvocation invocation) {
        // Access the command to get other option values
        MyCommand cmd = (MyCommand) invocation.getCommand();
        
        if (cmd.database != null) {
            // Complete tables based on selected database
            invocation.addAllCompleterValues(
                getTablesForDatabase(cmd.database)
            );
        }
    }
}

@CommandDefinition(name = "query")
public class MyCommand implements Command<CommandInvocation> {

    @Option(name = "database", description = "Database name")
    private String database;

    @Option(
        name = "table",
        completer = DatabaseTableCompleter.class,
        description = "Table name"
    )
    private String table;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        return CommandResult.SUCCESS;
    }
}
```

## Completer for Arguments

Works with `@Arguments` as well:

```java
@Arguments(completer = CommandNameCompleter.class)
private List<String> commandNames;
```

## Shell Completion Script Generation

Aesh can generate completion scripts for **bash**, **zsh**, and **fish** shells. There are two approaches:

- **Static scripts** — complete option names, subcommand names, and default values without a running JVM. Fast and zero-overhead, but cannot run custom `OptionCompleter` logic.
- **Dynamic callback scripts** — thin shell shims that call back to your program at tab-time via `--aesh-complete`, running the full aesh completion engine (including custom completers). Ideal for GraalVM native images where startup is near-instant.

### Static vs Dynamic

| | Static | Dynamic |
|---|---|---|
| Custom `OptionCompleter` support | No | Yes |
| JVM required at tab-time | No | Yes |
| Startup cost per Tab press | None | ~10ms (native) / ~200ms (JVM) |
| Best for | Option names, default values | Runtime-dependent completions |

### Supported Shells

| Shell | Generator | Script format |
|-------|-----------|---------------|
| Bash | `BashCompletionGenerator` | `complete -F` functions with `compgen -W` |
| Zsh | `ZshCompletionGenerator` | Native `compdef`/`_arguments` format |
| Fish | `FishCompletionGenerator` | `complete -c` commands with conditions |

### Static Completion Scripts

#### Quick Start with AeshRuntimeRunner

The simplest way to add static completion generation to your CLI tool:

```java
public class MyApp {
    public static void main(String[] args) {
        // Check for completion generation flag
        if (args.length > 0 && args[0].equals("--generate-completion")) {
            ShellType shell = ShellType.valueOf(args.length > 1 ? args[1] : "BASH");
            AeshRuntimeRunner.builder()
                    .command(MyCommand.class)
                    .generateCompletion(shell)
                    .execute();
            return;
        }

        // Normal execution
        AeshRuntimeRunner.builder()
                .command(MyCommand.class)
                .args(args)
                .execute();
    }
}
```

```bash
# Generate and install bash completion
$ myapp --generate-completion BASH > ~/.local/share/bash-completion/completions/myapp

# Generate zsh completion
$ myapp --generate-completion ZSH > ~/.zsh/completions/_myapp

# Generate fish completion
$ myapp --generate-completion FISH > ~/.config/fish/completions/myapp.fish
```

### One-Shot Static API

Generate a completion script from a command class without building a full runner:

```java
import org.aesh.util.completer.ShellCompletionGenerator;
import org.aesh.util.completer.ShellCompletionGenerator.ShellType;

String bashScript = ShellCompletionGenerator.generate(
        ShellType.BASH, MyCommand.class, "myapp");

String zshScript = ShellCompletionGenerator.generate(
        ShellType.ZSH, MyCommand.class, "myapp");

String fishScript = ShellCompletionGenerator.generate(
        ShellType.FISH, MyCommand.class, "myapp");
```

### Using the Strategy Interface

For more control, use the `ShellCompletionGenerator` interface directly with a command parser:

```java
import org.aesh.util.completer.ShellCompletionGenerator;

ShellCompletionGenerator generator = ShellCompletionGenerator.forShell(ShellType.BASH);
String script = generator.generate(parser, "myapp");
```

### What Gets Generated

The generators introspect the command model and produce:

- **Option names** (long and short forms)
- **Option aliases** (e.g., `--ea` for `--enableassertions`)
- **Negatable options** (e.g., `--no-verbose` for `--verbose`)
- **Default value completions** (offered when completing option values)
- **File path completion** for `File`, `Path`, and `Resource`-typed options
- **Subcommand names** for group commands
- **Positional argument completion** (file completion for `@Argument`/`@Arguments` with file types)

### Generated Script Examples

Given this command:

```java
@CommandDefinition(name = "deploy", description = "Deploy application")
public class DeployCommand implements Command<CommandInvocation> {

    @Option(shortName = 'e', defaultValue = {"dev", "staging", "prod"},
            description = "Target environment")
    private String environment;

    @Option(shortName = 'v', hasValue = false, negatable = true,
            description = "Verbose output")
    private boolean verbose;

    @Option(name = "config", aliases = {"cfg"}, description = "Config file")
    private File configFile;
}
```

**Bash** generates functions using `_init_completion`, `compgen -W` for values, and `_filedir` for file options.

**Zsh** generates native `_arguments` specs with exclusion groups, value completions, and `_files` for file-typed options.

**Fish** generates `complete -c` entries with `-l` (long), `-s` (short), `-r` (requires argument), `-a` (values), and `__fish_use_subcommand`/`__fish_seen_subcommand_from` conditions for group commands.

### Using the CompleterCommand

Aesh also provides a built-in `CompleterCommand` that generates completion scripts from a command class name:

```java
AeshRuntimeRunner.builder()
        .command(CompleterCommand.class)
        .args("--shell", "FISH", "com.example.MyCommand")
        .execute();
```

This writes the completion script to a file named `mycommand.fish` (or `.bash`/`.zsh`).

### Dynamic Callback Completion Scripts

Dynamic scripts generate thin shell shims that call back to your Java program at tab-time. When the user presses Tab, the shell invokes `myapp --aesh-complete -- <partial-command-line>`, and aesh's full completion engine runs — including all custom `OptionCompleter` implementations.

#### Quick Start

The simplest approach uses the `handleDynamicCompletion()` helper:

```java
public class MyApp {
    public static void main(String[] args) {
        // Handle dynamic completion requests (called by the shell script)
        if (AeshRuntimeRunner.handleDynamicCompletion(args, MyCommand.class)) {
            return;
        }

        // Check for completion script generation flag
        if (args.length > 0 && args[0].equals("--generate-completion")) {
            ShellType shell = ShellType.valueOf(args.length > 1 ? args[1] : "BASH");
            AeshRuntimeRunner.builder()
                    .command(MyCommand.class)
                    .generateDynamicCompletion(shell)
                    .execute();
            return;
        }

        // Normal execution
        AeshRuntimeRunner.builder()
                .command(MyCommand.class)
                .args(args)
                .execute();
    }
}
```

```bash
# Generate and install dynamic bash completion
$ myapp --generate-completion BASH > ~/.local/share/bash-completion/completions/myapp

# Generate dynamic zsh completion
$ myapp --generate-completion ZSH > ~/.zsh/completions/_myapp

# Generate dynamic fish completion
$ myapp --generate-completion FISH > ~/.config/fish/completions/myapp.fish
```

#### How It Works

1. You generate a dynamic completion script and install it in your shell
2. When the user presses Tab, the shell script calls `myapp --aesh-complete -- <partial-args>`
3. `handleDynamicCompletion()` detects the `--aesh-complete` flag, runs the completion engine, and prints one candidate per line to stdout
4. The shell script captures the output and presents it as completion candidates

#### One-Shot Dynamic API

Generate a dynamic script from a command class without building a runner:

```java
String bashScript = ShellCompletionGenerator.generateDynamic(
        ShellType.BASH, MyCommand.class, "myapp");

String zshScript = ShellCompletionGenerator.generateDynamic(
        ShellType.ZSH, MyCommand.class, "myapp");

String fishScript = ShellCompletionGenerator.generateDynamic(
        ShellType.FISH, MyCommand.class, "myapp");
```

#### GraalVM Native Images

Dynamic callback completion is ideal for GraalVM native images. Since native binaries start in ~10ms, tab completion feels instant. The generated scripts use the program name directly — no special configuration needed:

```bash
# Build native image (e.g., with Maven)
$ mvn package -Pnative

# Generate and install completion — the native binary handles --aesh-complete directly
$ ./target/myapp --generate-completion BASH > ~/.local/share/bash-completion/completions/myapp
```

For JVM-based tools where users run via `java -jar`, you can use a wrapper script and set `completionProgramName()` to match the wrapper name:

```java
AeshRuntimeRunner.builder()
        .command(MyCommand.class)
        .generateDynamicCompletion(ShellType.BASH)
        .completionProgramName("myapp")  // matches the wrapper script name
        .execute();
```
