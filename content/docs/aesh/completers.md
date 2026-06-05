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

Aesh can generate completion scripts for **bash**, **zsh**, and **fish** shells. Everything works automatically with the standard `AeshRuntimeRunner` pattern — no manual flag handling required.

### Quick Start

With the standard runner setup, your CLI tool automatically supports shell completion:

```java
public class MyApp {
    public static void main(String[] args) {
        AeshRuntimeRunner.builder()
                .command(MyCommand.class)
                .args(args)
                .execute();
    }
}
```

That's it. Your users can now generate and install completion scripts:

```bash
# Generate completion script (auto-detects shell from $SHELL)
$ myapp --aesh-completion

# Generate for a specific shell
$ myapp --aesh-completion bash
$ myapp --aesh-completion zsh
$ myapp --aesh-completion fish

# Auto-detect shell, generate, and install to the standard directory
$ myapp --aesh-completion-install
```

### Built-in Flags

| Flag | Purpose |
|------|---------|
| `--aesh-completion [bash\|zsh\|fish]` | Generate completion script to stdout (dynamic by default) |
| `--aesh-completion --static [bash\|zsh\|fish]` | Generate a static completion script (no JVM at tab-time) |
| `--aesh-completion-install` | Auto-detect shell, generate script, install with confirmation |
| `--aesh-complete -- <args>` | Runtime callback used by the generated shell scripts |

All flags are intercepted by `AeshRuntimeRunner.execute()` before your command runs, so they never reach your command's `execute()` method.

### Static vs Dynamic

By default, `--aesh-completion` generates **dynamic** scripts. Use `--static` for static scripts:

| | Static | Dynamic (default) |
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

### Installing Completions

The easiest way is `--aesh-completion-install`, which auto-detects your shell, generates the script, and installs it with confirmation:

```bash
$ myapp --aesh-completion-install
Write completion script to: /home/user/.bash_completion.d/myapp
Proceed? [y/N] y
Completion script installed to /home/user/.bash_completion.d/myapp
Note: 'myapp' must be on your $PATH for completions to work.
Restart your shell or source the file to activate completions.
```

Or install manually by redirecting the output:

```bash
# Bash
$ myapp --aesh-completion bash > ~/.local/share/bash-completion/completions/myapp

# Zsh
$ myapp --aesh-completion zsh > ~/.zsh/completions/_myapp

# Fish
$ myapp --aesh-completion fish > ~/.config/fish/completions/myapp.fish
```

### How Dynamic Completion Works

1. You generate a completion script and install it in your shell (via `--aesh-completion-install` or manual redirect)
2. When the user presses Tab, the shell script calls `myapp --aesh-complete -- <partial-args>`
3. Aesh's completion engine runs (including custom `OptionCompleter` implementations) and prints candidates to stdout
4. The shell presents the candidates as completion suggestions

### What Gets Generated

The generators introspect the command model and produce:

- **Option names** (long and short forms)
- **Option aliases** (e.g., `--ea` for `--enableassertions`)
- **Negatable options** (e.g., `--no-verbose` for `--verbose`)
- **Default value completions** (offered when completing option values)
- **File path completion** for `File`, `Path`, and [`Resource`](../resources)-typed options
- **Subcommand names** for group commands
- **Positional argument completion** (file completion for `@Argument`/`@Arguments` with file types)

### Completion Fallback Control

By default, when no completion candidates are available for an argument, the shell falls back to file/path completion for `String`, `File`, and `Path` types, and no fallback for enum types. You can override this per-argument using `completeFallback`:

```java
@Argument(description = "Script file")
String script;  // DEFAULT: auto-detected as FILES (String type)

@Argument(description = "Catalog name", completeFallback = CompletionFallback.NONE)
String catalog;  // No file fallback -- only completer candidates

@Option(name = "workspace", completeFallback = CompletionFallback.DIRECTORIES)
String workspace;  // Only directories, no regular files
```

| Value | Behavior |
|-------|----------|
| `DEFAULT` | Auto-detect: `FILES` for String/File/Path, `NONE` for enums |
| `FILES` | Offer file and directory path completion |
| `DIRECTORIES` | Offer only directory completion (no regular files) |
| `NONE` | No fallback -- only show candidates from completers |

This is especially useful for commands where some arguments expect file paths and others expect non-file values like identifiers, versions, or names:

```java
@CommandDefinition(name = "jdk", description = "Manage JDKs",
        groupCommands = { JdkInstallCommand.class })
public class JdkCommand implements Command<CommandInvocation> { }

@CommandDefinition(name = "install", description = "Install a JDK version")
public class JdkInstallCommand implements Command<CommandInvocation> {
    @Argument(description = "JDK version", completeFallback = CompletionFallback.NONE)
    String version;  // "17", "21" -- not a file path
}
```

### GraalVM Native Images

Dynamic callback completion is ideal for GraalVM native images. Since native binaries start in ~10ms, tab completion feels instant:

```bash
# Build native image
$ mvn package -Pnative

# Install completion — the native binary handles --aesh-complete directly
$ ./target/myapp --aesh-completion-install
```

For JVM-based tools where users run via `java -jar`, use a wrapper script and set `completionProgramName()`:

```java
AeshRuntimeRunner.builder()
        .command(MyCommand.class)
        .completionProgramName("myapp")  // matches the wrapper script name
        .args(args)
        .execute();
```

### Programmatic API

For advanced use cases, you can generate scripts programmatically:

```java
import org.aesh.util.completer.ShellCompletionGenerator;
import org.aesh.util.completer.ShellCompletionGenerator.ShellType;

// Static scripts
String bashScript = ShellCompletionGenerator.generate(
        ShellType.BASH, MyCommand.class, "myapp");

// Dynamic scripts
String fishScript = ShellCompletionGenerator.generateDynamic(
        ShellType.FISH, MyCommand.class, "myapp");

// Using the strategy interface for more control
ShellCompletionGenerator generator = ShellCompletionGenerator.forShell(ShellType.ZSH);
String script = generator.generate(parser, "myapp");
```
