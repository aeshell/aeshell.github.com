---
date: '2026-09-04T12:00:00+02:00'
draft: false
title: 'Autosuggestions'
weight: 8
---

Æsh Readline supports fish-style autosuggestions: as you type, the most recent matching history entry appears as greyed-out ghost text after the cursor. Press Right arrow to accept the full suggestion, or Ctrl+Right to accept one character at a time.

## Enabling Autosuggestions

### Via ReadlineRequest (recommended)

The simplest way to enable autosuggestions is through the `ReadlineRequest` builder:

```java
readline.readline(ReadlineRequest.builder()
        .connection(connection)
        .prompt(new Prompt("$ "))
        .requestHandler(this::handleInput)
        .historySuggestions(true)
        .build());
```

This activates `HistorySuggestionProvider`, which searches the command history for the most recent entry matching the current input prefix.

### Via Readline directly

For finer control, call `enableHistorySuggestions()` on the `Readline` instance:

```java
Readline readline = ReadlineBuilder.builder().build();
readline.enableHistorySuggestions();
```

Or provide a custom `SuggestionProvider`:

```java
readline.setSuggestionProvider(new MySuggestionProvider());
```

## Accepting Suggestions

| Key | Action |
|-----|--------|
| `Right` (`Ctrl+F`) | Accept entire suggestion |
| `End` (`Ctrl+E`) | Accept entire suggestion |
| `Alt+F` | Accept next word |
| `Ctrl+Right` | Accept next character |

Any other keystroke clears the ghost text and processes the key normally. If you continue typing and the input still matches a history entry, a new suggestion appears.

## How It Works

After each keystroke, the suggestion provider receives the current buffer contents and returns the suffix to display. For history suggestions, this means:

1. The provider searches history (most recent first) for an entry starting with the current input.
2. If found, it returns the portion of that entry after the current input.
3. The suffix is rendered as dim (SGR 2) text after the cursor, without modifying the buffer.
4. The cursor stays at the end of the actual input — the ghost text is purely visual.

When you accept (fully or partially), the accepted characters are inserted into the buffer as real input, and any remaining ghost text is re-rendered.

## Custom Ghost Text Styling

By default, ghost text uses SGR 2 (dim/faint). Some terminals render dim text poorly — you can change the style:

```java
readline.readline(ReadlineRequest.builder()
        .connection(connection)
        .prompt(new Prompt("$ "))
        .requestHandler(this::handleInput)
        .historySuggestions(true)
        .ghostTextStyle("\033[90m", "\033[39m")  // bright black (grey)
        .build());
```

Common style pairs:

| Style | On | Off | Notes |
|-------|----|-----|-------|
| Dim (default) | `\033[2m` | `\033[22m` | Standard SGR 2; some terminals render poorly |
| Grey | `\033[90m` | `\033[39m` | Bright black foreground; works on most terminals |
| Dim + italic | `\033[2;3m` | `\033[22;23m` | Combined dim and italic |
| Custom color | `\033[38;5;240m` | `\033[39m` | 256-color grey (index 240) |

You can also set the style programmatically on the `AeshConsoleBuffer`:

```java
AeshConsoleBuffer buffer = ...;
buffer.setGhostTextStyle("\033[90m", "\033[39m");
```

## Custom Suggestion Providers

Implement `SuggestionProvider` to provide suggestions from any source:

```java
import org.aesh.readline.SuggestionProvider;

public class FileSuggestionProvider implements SuggestionProvider {
    @Override
    public String suggest(String buffer) {
        // Return the suffix to display, or null for no suggestion
        Path dir = Paths.get(".");
        try (DirectoryStream<Path> stream = Files.newDirectoryStream(dir, buffer + "*")) {
            for (Path entry : stream) {
                String name = entry.getFileName().toString();
                return name.substring(buffer.length());
            }
        } catch (IOException e) {
            return null;
        }
        return null;
    }
}
```

### Chaining Providers

Use `CompositeSuggestionProvider` to try multiple providers in order:

```java
readline.setSuggestionProvider(
        new CompositeSuggestionProvider(
                new HistorySuggestionProvider(history),
                new FileSuggestionProvider()
        ));
```

The composite provider returns the first non-null suggestion. In this example, history matches take priority over file matches.

## Relationship with Ctrl+R

Autosuggestions and [fuzzy history search](fuzzy-history-search) are complementary:

| Feature | Autosuggestions | Fuzzy Search (Ctrl+R) |
|---------|----------------|----------------------|
| Trigger | Automatic as you type | Manual (Ctrl+R) |
| Matching | Prefix match (most recent) | Fuzzy match (ranked) |
| Results | Single suggestion | Scrollable list |
| Use case | Repeating recent commands | Finding commands you half-remember |

Both features use the same history — enabling one does not disable the other.
