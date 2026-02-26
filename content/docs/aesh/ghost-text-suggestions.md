---
date: '2026-02-26T10:00:00+01:00'
draft: false
title: 'Ghost Text Suggestions'
weight: 9
---

Ghost text suggestions display inline, dimmed text ahead of the cursor as you type, showing what Æsh predicts you want to enter. Unlike [tab completion](completers), which requires pressing Tab, ghost text appears automatically and can be accepted by pressing the right arrow key.

## How It Works

As you type, ghost text suggestions appear automatically:

```
myapp> ca|che                    ← "che" appears dimmed after cursor
myapp> connect --ho|st=          ← "st=" appears dimmed after cursor
myapp> git co|mmit               ← "mmit" appears dimmed after cursor
```

- **Right arrow** accepts the suggestion
- **Keep typing** to narrow or dismiss the suggestion
- Only shown when there is exactly one unambiguous match

## CommandSuggestionProvider

`CommandSuggestionProvider` uses the command registry to suggest command names, subcommand names, and option names as you type. It implements the `SuggestionProvider` interface from Æsh Readline.

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

You can combine multiple suggestion sources using `CompositeSuggestionProvider`. The first provider that returns a non-null suggestion wins. This lets you layer history-based suggestions with command-based suggestions:

```java
import org.aesh.readline.CompositeSuggestionProvider;
import org.aesh.readline.SuggestionProvider;

// History-based suggestions (checked first)
SuggestionProvider historyProvider = buffer -> {
    // Return matching history entry suffix, or null
    return findHistoryMatch(buffer);
};

// Command registry suggestions (fallback)
CommandSuggestionProvider<?> commandProvider =
        new CommandSuggestionProvider<>(console.getCommandRegistry());

// Combine: history takes priority, falls back to commands
CompositeSuggestionProvider composite =
        new CompositeSuggestionProvider(historyProvider, commandProvider);

console.setSuggestionProvider(composite);
```

## Custom SuggestionProvider

You can implement the `SuggestionProvider` interface directly for custom suggestion logic:

```java
import org.aesh.readline.SuggestionProvider;

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
