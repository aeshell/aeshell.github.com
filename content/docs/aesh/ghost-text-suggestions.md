---
date: '2026-02-26T10:00:00+01:00'
draft: false
title: 'Ghost Text Suggestions'
weight: 9
---

Ghost text suggestions display inline, dimmed text ahead of the cursor as you type, showing what Aesh predicts you want to enter. Unlike [tab completion](completers), which requires pressing Tab, ghost text appears automatically and can be accepted with keyboard shortcuts.

## How It Works

As you type, ghost text suggestions appear automatically:

```
myapp> ca|che                    ← "che" appears dimmed after cursor
myapp> connect --ho|st=          ← "st=" appears dimmed after cursor
myapp> mvn clean tes|t -pl aesh  ← "t -pl aesh" from history, dimmed
```

- **Right arrow / End** -- Accept the full suggestion
- **Alt+F / Ctrl+Right** -- Accept the next word from the suggestion
- **Keep typing** -- Narrow or dismiss the suggestion
- Ghost text updates automatically after every keystroke, including backspace

## History Suggestions

Aesh Readline includes a built-in `HistorySuggestionProvider` that suggests commands from your history as you type, similar to [fish shell's auto-suggestions](https://fishshell.com/docs/current/interactive.html#autosuggestions). It searches history from most recent to oldest, finding the first entry that starts with what you've typed, and shows the remaining text as ghost text.

```
myapp> mvn c|lean test -pl aesh -Dtest=ProcessorTest
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ from history
```

### Automatic activation

When you call `setSuggestionProvider()` on `ReadlineConsole`, history suggestions are automatically enabled and take priority over command-level suggestions. No additional setup is needed -- if the current input matches a history entry, that suggestion is shown; otherwise, the command suggestion provider is used as a fallback.

### Standalone history suggestions

If you want history suggestions without a command-level provider (e.g., in a plain readline application):

```java
Readline readline = new Readline(editMode, history, completionHandler);
readline.enableHistorySuggestions();
```

### How it works

- Searches `History.getAll()` in reverse order (most recent first)
- Uses prefix matching -- the buffer must match the beginning of a history entry
- Returns only the suffix (the part you haven't typed yet)
- Combined with other providers via `CompositeSuggestionProvider` (history first)

## CommandSuggestionProvider

`CommandSuggestionProvider` uses the command registry to suggest command names, subcommand names, and option names as you type. It implements the `SuggestionProvider` interface from Aesh Readline.

### What It Suggests

| Context | Example Input | Suggestion |
|---------|--------------|------------|
| Command names | `ca` | `che` (from `cache`) |
| Subcommand names | `git co` | `mmit` (from `commit`) |
| Option names | `connect --ho` | `st=` (from `--host`) |

Suggestions are only shown when there is a single unambiguous match. If the input `c` could match both `cache` and `connect`, no suggestion is shown.

For options that accept a value, the suggestion appends `=` after the option name. Boolean options (flags) do not get the trailing `=`.

### Setup

Create a `CommandSuggestionProvider` from your command registry and attach it to the console:

```java
import org.aesh.command.impl.completer.CommandSuggestionProvider;
import org.aesh.console.ReadlineConsole;

ReadlineConsole console = new ReadlineConsole(settings);

// Create suggestion provider from the console's registry
CommandSuggestionProvider<?> provider =
        new CommandSuggestionProvider<>(console.getCommandRegistry());

console.setSuggestionProvider(provider);
console.start();
```

### With AeshConsoleRunner

If you use `AeshConsoleRunner`, you can access the underlying `ReadlineConsole` to set up suggestions:

```java
ReadlineConsole console = new ReadlineConsole(settings);
console.setPrompt(new Prompt("myapp> "));
console.setSuggestionProvider(
        new CommandSuggestionProvider<>(console.getCommandRegistry()));
console.start();
```

## CompositeSuggestionProvider

Multiple suggestion sources are combined using `CompositeSuggestionProvider`. The first provider that returns a non-null suggestion wins.

{{< callout type="info" >}}
When you call `setSuggestionProvider()` with history available, Aesh automatically creates a `CompositeSuggestionProvider` with history suggestions first and your provider second. You typically don't need to create one manually.
{{< /callout >}}

For custom chaining:

```java
import org.aesh.readline.suggestion.CompositeSuggestionProvider;
import org.aesh.readline.suggestion.SuggestionProvider;

SuggestionProvider myProvider = buffer -> {
    // Custom suggestion logic
    return null;
};

CommandSuggestionProvider<?> commandProvider =
        new CommandSuggestionProvider<>(console.getCommandRegistry());

// Custom ordering: your provider first, then commands
CompositeSuggestionProvider composite =
        new CompositeSuggestionProvider(myProvider, commandProvider);

console.setSuggestionProvider(composite);
```

## Custom SuggestionProvider

You can implement the `SuggestionProvider` interface directly for custom suggestion logic:

```java
import org.aesh.readline.suggestion.SuggestionProvider;

SuggestionProvider myProvider = buffer -> {
    // Return the suffix to append as ghost text, or null for no suggestion
    if (buffer.startsWith("hel")) {
        return "lo";
    }
    return null;
};

console.setSuggestionProvider(myProvider);
```

The `suggest` method receives the current input buffer and returns the text to display after the cursor as ghost text. Return `null` when there is no suggestion.

## Ghost Text vs Tab Completion

| Feature | Ghost Text | Tab Completion |
|---------|-----------|----------------|
| **Trigger** | Automatic as you type | Press Tab |
| **Display** | Inline dimmed text | List below prompt |
| **Matches** | Single unambiguous only | Shows all matches |
| **Accept** | Right arrow | Tab / Enter |
| **Scope** | Commands, subcommands, options | Options, arguments, custom values |

Both features complement each other. Ghost text gives fast inline suggestions for unambiguous matches, while tab completion helps explore all available options when there are multiple possibilities.
