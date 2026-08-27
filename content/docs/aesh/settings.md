---
date: '2026-01-26T10:00:00+01:00'
draft: false
title: 'Settings Configuration'
weight: 15
---

The `Settings` class provides comprehensive configuration for Æsh console applications. It controls behavior for history, aliases, editing modes, logging, and many other aspects of the shell environment.

## Overview

Settings are configured using the builder pattern and passed to either `AeshConsoleRunner` or `AeshRuntimeRunner`:

```java
import org.aesh.command.settings.Settings;
import org.aesh.command.settings.SettingsBuilder;

Settings settings = SettingsBuilder.builder()
        .enableHistory(true)
        .historyFile(new File(".myapp_history"))
        .historySize(500)
        .enableAlias(true)
        .build();

AeshConsoleRunner.builder()
        .settings(settings)
        .command(MyCommand.class)
        .start();
```

## SettingsBuilder API

### Creating a Builder

```java
// Default settings
SettingsBuilder builder = SettingsBuilder.builder();

// Copy from existing settings
SettingsBuilder builder = SettingsBuilder.builder(existingSettings);
```

### Building Settings

```java
Settings settings = builder.build();
```

## Configuration Options

### History Configuration

#### enableHistory(boolean)

Enables or disables command history. Default: `true`

```java
SettingsBuilder.builder()
        .enableHistory(true)
        .build();
```

#### historyFile(File)

Sets the file for persisting command history. Default: `null` (in-memory only)

```java
SettingsBuilder.builder()
        .historyFile(new File(System.getProperty("user.home"), ".myapp_history"))
        .build();
```

#### historySize(int)

Sets the maximum number of history entries. Default: `500`

```java
SettingsBuilder.builder()
        .historySize(1000)
        .build();
```

#### historyPersistent(boolean)

Controls whether history is persisted to file. Default: `true` (if historyFile is set)

```java
SettingsBuilder.builder()
        .historyPersistent(true)
        .build();
```

#### historyDisabled(boolean)

Completely disables history functionality. Default: `false`

```java
SettingsBuilder.builder()
        .historyDisabled(false)
        .build();
```

### Alias Configuration

#### enableAlias(boolean)

Enables command aliases. Default: `false`

```java
SettingsBuilder.builder()
        .enableAlias(true)
        .build();
```

#### aliasFile(File)

Sets the file for persisting aliases. Default: `null`

```java
SettingsBuilder.builder()
        .aliasFile(new File(System.getProperty("user.home"), ".myapp_aliases"))
        .build();
```

#### persistAlias(boolean)

Controls whether aliases are persisted to file. Default: `true` (if aliasFile is set)

```java
SettingsBuilder.builder()
        .persistAlias(true)
        .build();
```

### Export Configuration

#### enableExport(boolean)

Enables export functionality for environment variables. Default: `false`

```java
SettingsBuilder.builder()
        .enableExport(true)
        .build();
```

#### exportFile(File)

Sets the file for persisting exported variables. Default: `null`

```java
SettingsBuilder.builder()
        .exportFile(new File(System.getProperty("user.home"), ".myapp_exports"))
        .build();
```

#### persistExport(boolean)

Controls whether exports are persisted to file. Default: `true` (if exportFile is set)

```java
SettingsBuilder.builder()
        .persistExport(true)
        .build();
```

### Editing Mode

#### mode(EditMode.Mode)

Sets the editing mode (Emacs or Vi). Default: `EditMode.Mode.EMACS`

```java
import org.aesh.readline.editing.EditMode;

SettingsBuilder.builder()
        .mode(EditMode.Mode.EMACS)  // or EditMode.Mode.VI
        .build();
```

### Logging

#### logging(boolean)

Enables internal logging. Default: `false`

```java
SettingsBuilder.builder()
        .logging(true)
        .build();
```

#### logFile(String)

Sets the log file path. Default: `null`

```java
SettingsBuilder.builder()
        .logging(true)
        .logFile("/var/log/myapp/aesh.log")
        .build();
```

### Command Registry

#### commandRegistry(CommandRegistry)

Sets a custom command registry.

```java
import org.aesh.command.registry.MutableCommandRegistry;

MutableCommandRegistry registry = new MutableCommandRegistry();
registry.addCommand(MyCommand.class);

SettingsBuilder.builder()
        .commandRegistry(registry)
        .build();
```

### Connection and Terminal

#### connection(Connection)

Sets a custom connection for terminal I/O.

```java
SettingsBuilder.builder()
        .connection(myConnection)
        .build();
```

#### inputStream(InputStream)

Sets the input stream. Default: `System.in`

```java
SettingsBuilder.builder()
        .inputStream(new FileInputStream("input.txt"))
        .build();
```

#### outputStream(PrintStream)

Sets the output stream. Default: `System.out`

```java
SettingsBuilder.builder()
        .outputStream(new PrintStream(new FileOutputStream("output.txt")))
        .build();
```

#### outputStreamError(PrintStream)

Sets the error output stream. Default: `System.err`

```java
SettingsBuilder.builder()
        .outputStreamError(new PrintStream(new FileOutputStream("error.txt")))
        .build();
```

### Operators and Redirection

#### enableOperatorParser(boolean)

Enables parsing of command operators (`|`, `>`, `>>`, `&&`, `||`, `;`). Default: `true`

```java
SettingsBuilder.builder()
        .enableOperatorParser(true)
        .build();
```

#### setPipe(boolean)

Enables pipe (`|`) operator support. Default: `true`

```java
SettingsBuilder.builder()
        .setPipe(true)
        .build();
```

#### setRedirection(boolean)

Enables redirection (`>`, `>>`, `<`) support. Default: `true`

```java
SettingsBuilder.builder()
        .setRedirection(true)
        .build();
```

### Command Invocation

#### commandInvocationProvider(CommandInvocationProvider)

Sets a custom `CommandInvocation` provider for dependency injection.

```java
SettingsBuilder.builder()
        .commandInvocationProvider(new MyCommandInvocationProvider())
        .build();
```

See [Custom Command Invocation](/docs/aesh/command-invocation#custom-commandinvocation) for details.

### Completion

#### completionHandler(CompletionHandler)

Sets a custom completion handler.

```java
SettingsBuilder.builder()
        .completionHandler(new MyCompletionHandler())
        .build();
```

### Sub-Command Mode Settings

#### subCommandModeSettings(SubCommandModeSettings)

Configures the behavior of sub-command mode for group commands.

```java
import org.aesh.command.settings.SubCommandModeSettings;

SubCommandModeSettings subCmdSettings = SubCommandModeSettings.builder()
        .enabled(true)                          // Enable/disable sub-command mode
        .exitCommand("exit")                    // Primary exit command
        .alternativeExitCommand("..")           // Alternative exit command
        .contextSeparator(":")                  // Nested context separator
        .showArgumentInPrompt(true)             // Show value in prompt
        .contextCommand("context")              // Command to display context
        .enterMessage("Entering {name} mode.")  // Entry message
        .exitHint("Type '{exit}' to return.")   // Exit hint
        .exitOnCtrlC(true)                      // Ctrl+C exits sub-command mode
        .build();

SettingsBuilder.builder()
        .subCommandModeSettings(subCmdSettings)
        .build();
```

See [Sub-Command Mode](/docs/aesh/sub-command-mode) for complete documentation.

### Ghost Text and Tail Tips

#### tailTipSuggestions(boolean)

Enables tail tip suggestions -- dimmed parameter hints shown after the cursor in interactive mode. When enabled, remaining options and arguments are displayed as ghost text after each completed word. Only applies to `ReadlineConsole` (REPL mode), not `AeshRuntimeRunner`. Default: `false`

```java
SettingsBuilder.builder()
        .tailTipSuggestions(true)
        .build();
```

See [Ghost Text Suggestions](../ghost-text-suggestions#tailtipsuggestionprovider) for details on how tail tips work and how they interact with other suggestion providers.

### Dynamic Prompt

#### promptSupplier(Supplier&lt;Prompt&gt;)

Set a dynamic prompt supplier that is called before each readline cycle. Enables prompts that change based on context (e.g., git branch, command duration, current directory). If set, takes precedence over a static prompt.

```java
SettingsBuilder.builder()
        .promptSupplier(() -> Prompt.builder()
                .line("myapp on " + getGitBranch())
                .line("> ")
                .rightPrompt(LocalTime.now().format(DateTimeFormatter.ofPattern("HH:mm")))
                .build())
        .build();
```

Multi-line prompts are created with `Prompt.builder().line("...").line("...").build()`. The cursor math uses only the last line's length. Right prompts are displayed right-aligned and disappear when input grows too long.

### Command Execution Listener

#### commandExecutionListener(CommandExecutionListener)

Set a callback that fires after each command finishes execution. The listener receives the full command line, the `CommandResult`, and the wall-clock execution time in milliseconds.

```java
SettingsBuilder.builder()
        .commandExecutionListener((commandLine, result, durationMs) -> {
            System.out.println("Command '" + commandLine + "' completed in " + durationMs + "ms");
        })
        .build();
```

The listener also fires for pre-Process errors (command not found, parse errors), giving consumers a complete view of all command outcomes.

To receive the exception that caused a failure, override the 4-arg default method:

```java
SettingsBuilder.builder()
        .commandExecutionListener(new CommandExecutionListener() {
            @Override
            public void onCommandComplete(String commandLine, CommandResult result, long durationMs) {}

            @Override
            public void onCommandComplete(String commandLine, CommandResult result,
                    long durationMs, Throwable error) {
                if (error != null) {
                    System.err.println("Command failed: " + error.getMessage());
                }
            }
        })
        .build();
```

This combines well with `promptSupplier` for starship-style prompts that show the last command's duration:

```java
AtomicLong lastDuration = new AtomicLong();

SettingsBuilder.builder()
        .commandExecutionListener((line, result, ms) -> lastDuration.set(ms))
        .promptSupplier(() -> {
            long ms = lastDuration.getAndSet(0);
            String dur = ms > 1000 ? String.format(" took %.1fs", ms / 1000.0) : "";
            return Prompt.builder()
                    .line("myapp" + dur)
                    .line("> ")
                    .build();
        })
        .build();
```

#### commandOutputHandler(Consumer&lt;String&gt;)

Capture what commands write via `invocation.println()` / `invocation.print()`, separately from readline chrome, prompt drawing, and ANSI escape sequences. Output is teed to both the terminal and the handler. When not set (the default), no overhead is added to the output path.

```java
StringBuilder capturedOutput = new StringBuilder();

SettingsBuilder.builder()
        .commandOutputHandler(capturedOutput::append)
        .build();
```

This is primarily useful for test frameworks that need clean command output without parsing the full terminal buffer.

### Additional Options

#### readInputrc(boolean)

Controls whether to read the `.inputrc` file for readline configuration. Default: `true`

```java
SettingsBuilder.builder()
        .readInputrc(true)
        .build();
```

#### parseOperators(boolean)

Controls operator parsing. Default: `true`

```java
SettingsBuilder.builder()
        .parseOperators(true)
        .build();
```

#### echoCtrl(boolean)

Controls whether control characters are echoed. Default: `true`

```java
SettingsBuilder.builder()
        .echoCtrl(true)
        .build();
```

#### setInterruptHandler(Consumer&lt;Void&gt;)

Set a custom handler that is called when Ctrl-C is pressed **outside** of command execution (i.e., at the prompt). During command execution, Ctrl-C is handled by the command's thread interrupt mechanism (see [Handling Ctrl-C](../command-invocation/#handling-ctrl-c-interrupts)).

```java
SettingsBuilder.builder()
        .setInterruptHandler(v -> System.out.println("Interrupted at prompt"))
        .build();
```

#### redrawPromptOnInterrupt(boolean)

Controls whether the prompt is redrawn after Ctrl-C at the prompt. Default: `false`

```java
SettingsBuilder.builder()
        .redrawPromptOnInterrupt(true)
        .build();
```

## Complete Example

```java
import org.aesh.command.settings.Settings;
import org.aesh.command.settings.SettingsBuilder;
import org.aesh.readline.editing.EditMode;
import java.io.File;

public class ConfiguredConsole {
    public static void main(String[] args) {
        String userHome = System.getProperty("user.home");
        
        Settings settings = SettingsBuilder.builder()
                // History configuration
                .enableHistory(true)
                .historyFile(new File(userHome, ".myapp_history"))
                .historySize(1000)
                .historyPersistent(true)
                
                // Alias configuration
                .enableAlias(true)
                .aliasFile(new File(userHome, ".myapp_aliases"))
                
                // Export configuration
                .enableExport(true)
                .exportFile(new File(userHome, ".myapp_exports"))
                
                // Editing mode
                .mode(EditMode.Mode.EMACS)
                
                // Logging
                .logging(true)
                .logFile("/tmp/myapp-aesh.log")
                
                // Operators
                .enableOperatorParser(true)
                .setPipe(true)
                .setRedirection(true)
                
                .build();
        
        AeshConsoleRunner.builder()
                .settings(settings)
                .command(MyCommand.class)
                .command(OtherCommand.class)
                .addExitCommand()
                .prompt("[myapp]$ ")
                .start();
    }
}
```

## Accessing Settings at Runtime

Commands can access settings through the `CommandInvocationConfiguration`:

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    CommandInvocationConfiguration config = invocation.getConfiguration();
    // Access configuration properties...
    return CommandResult.SUCCESS;
}
```

## Settings Summary Table

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `enableHistory` | `boolean` | `true` | Enable command history |
| `historyFile` | `File` | `null` | History persistence file |
| `historySize` | `int` | `500` | Maximum history entries |
| `historyPersistent` | `boolean` | `true` | Persist history to file |
| `historyDisabled` | `boolean` | `false` | Completely disable history |
| `enableAlias` | `boolean` | `false` | Enable command aliases |
| `aliasFile` | `File` | `null` | Alias persistence file |
| `persistAlias` | `boolean` | `true` | Persist aliases to file |
| `enableExport` | `boolean` | `false` | Enable environment exports |
| `exportFile` | `File` | `null` | Export persistence file |
| `persistExport` | `boolean` | `true` | Persist exports to file |
| `mode` | `EditMode.Mode` | `EMACS` | Editing mode (EMACS or VI) |
| `logging` | `boolean` | `false` | Enable internal logging |
| `logFile` | `String` | `null` | Log file path |
| `enableOperatorParser` | `boolean` | `true` | Parse command operators |
| `setPipe` | `boolean` | `true` | Enable pipe operator |
| `setRedirection` | `boolean` | `true` | Enable redirection operators |
| `readInputrc` | `boolean` | `true` | Read .inputrc configuration |
| `echoCtrl` | `boolean` | `true` | Echo control characters |
| `tailTipSuggestions` | `boolean` | `false` | Show parameter hints after cursor |
| `promptSupplier` | `Supplier<Prompt>` | `null` | Dynamic prompt called before each readline cycle |
| `commandExecutionListener` | `CommandExecutionListener` | `null` | Callback after each command execution (and pre-Process errors) with duration and optional error |
| `commandOutputHandler` | `Consumer<String>` | `null` | Captures command output separately from readline chrome; zero overhead when null |
| `subCommandModeSettings` | `SubCommandModeSettings` | defaults | Sub-command mode configuration |

## Best Practices

1. **Always set history file** - For production applications, always configure a history file so users don't lose their command history between sessions.

2. **Use user home for config files** - Store configuration files in the user's home directory following platform conventions.

3. **Enable aliases for power users** - Aliases let users create shortcuts for frequently used commands.

4. **Consider Vi mode** - Some users prefer Vi editing mode. Consider making this configurable.

5. **Log during development** - Enable logging during development to debug issues.

6. **Disable operators when not needed** - If your application doesn't support piping or redirection, disable them to prevent confusion.

```java
// Development settings
Settings devSettings = SettingsBuilder.builder()
        .logging(true)
        .logFile("aesh-debug.log")
        .build();

// Production settings
Settings prodSettings = SettingsBuilder.builder()
        .enableHistory(true)
        .historyFile(new File(System.getProperty("user.home"), ".myapp_history"))
        .historySize(1000)
        .enableAlias(true)
        .build();
```
