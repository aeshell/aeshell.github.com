---
date: '2026-01-26T10:00:00+01:00'
draft: false
title: 'Readline API Reference'
weight: 3
---

The `Readline` class is the core API for reading input from terminals. It provides line editing, history, completion, and event handling capabilities.

## Overview

```java
import org.aesh.readline.Readline;
import org.aesh.readline.ReadlineBuilder;

Readline readline = ReadlineBuilder.builder()
        .enableHistory(true)
        .build();

readline.readline(connection, "prompt> ", input -> {
    if (input != null) {
        processInput(input);
    }
});
```

## ReadlineBuilder

### Builder Methods

| Method | Type | Default | Description |
|--------|------|---------|-------------|
| `editMode(EditMode)` | `EditMode` | Emacs | Set editing mode (Emacs or Vi) |
| `history(History)` | `History` | InMemoryHistory | Custom history implementation |
| `enableHistory(boolean)` | `boolean` | `true` | Enable/disable history |
| `historySize(int)` | `int` | `500` | Maximum history entries |
| `historyFile(String)` | `String` | `null` | History persistence file |
| `completionHandler(CompletionHandler)` | `CompletionHandler` | Default | Custom completion handler |
| `connection(Connection)` | `Connection` | `null` | Default connection |
| `inputrcFile(String)` | `String` | `~/.inputrc` | Readline configuration file |
| `parseInputrc(boolean)` | `boolean` | `true` | Parse inputrc file |
| `enableBracketedPaste(boolean)` | `boolean` | `true` | Enable bracketed paste mode |

### Creating a Readline Instance

```java
// Minimal configuration
Readline readline = ReadlineBuilder.builder().build();

// Full configuration
Readline readline = ReadlineBuilder.builder()
        .editMode(EditModeBuilder.builder()
                .mode(EditMode.Mode.EMACS)
                .create())
        .history(new FileHistory(new File(".history"), 500, true))
        .enableHistory(true)
        .historySize(1000)
        .completionHandler(new SimpleCompletionHandler())
        .enableBracketedPaste(true)
        .parseInputrc(true)
        .build();
```

## Readline Methods

### readline() - Basic

Read a line of input with a string prompt:

```java
readline.readline(connection, "$ ", input -> {
    // input is the string entered by the user
    // input is null if the user pressed Ctrl-D (EOF)
});
```

**Parameters:**
- `connection` - The terminal connection
- `prompt` - String prompt to display
- `requestHandler` - Callback for when input is received

### readline() - With Prompt Object

Read input with a styled `Prompt`:

```java
Prompt prompt = new Prompt(new TerminalString("$ ", Color.GREEN));

readline.readline(connection, prompt, input -> {
    // Handle input
});
```

### readline() - With Completions

Provide tab completions for the current input:

```java
List<Completion> completions = Arrays.asList(
        new Completion("help", "Display help"),
        new Completion("exit", "Exit the shell"),
        new Completion("list", "List items")
);

readline.readline(connection, prompt, completions, input -> {
    // Handle input
});
```

### readline() - With Pre-processors

Add input pre-processors that can transform input before it's returned:

```java
List<Function<String, Optional<String>>> preProcessors = Arrays.asList(
        // Trim whitespace
        input -> Optional.of(input.trim()),
        
        // Expand aliases
        input -> {
            if (input.equals("ll")) {
                return Optional.of("ls -l");
            }
            return Optional.of(input);
        },
        
        // Handle empty input
        input -> input.isEmpty() ? Optional.empty() : Optional.of(input)
);

readline.readline(connection, prompt, completions, preProcessors, input -> {
    // input has been processed by all pre-processors
});
```

### readline() - With Custom History

Override the default history for this read operation:

```java
History customHistory = new InMemoryHistory(100);

readline.readline(connection, prompt, completions, preProcessors, customHistory, input -> {
    // This read uses customHistory instead of the default
});
```

### readline() - With Cursor Listener

Get callbacks when the cursor position changes:

```java
CursorListener listener = new CursorListener() {
    @Override
    public void cursorMoved(int oldPos, int newPos) {
        // Cursor moved from oldPos to newPos
    }
    
    @Override
    public void inputChanged(String buffer) {
        // Input buffer content changed
    }
};

readline.readline(connection, prompt, completions, preProcessors, history, listener, input -> {
    // Handle input
});
```

### readline() - Full Signature

The complete signature with all options:

```java
readline.readline(
        Connection conn,           // Terminal connection
        Prompt prompt,             // Prompt to display
        Consumer<String> handler,  // Input handler callback
        List<Completion> completions,  // Tab completions
        List<Function<String, Optional<String>>> preProcessors,  // Input transformers
        History history,           // Custom history
        CursorListener listener,   // Cursor events
        EnumMap<ReadlineFlag, Integer> flags  // Behavior flags
);
```

### ReadlineFlag

Control readline behavior with flags:

| Flag | Description |
|------|-------------|
| `NO_PROMPT_REDRAW_ON_INTR` | Don't redraw prompt after interrupt (Ctrl-C) |
| `NO_SYNCHRONIZED_OUTPUT` | Disable automatic synchronized output (Mode 2026) |
| `NO_GRAPHEME_CLUSTER_MODE` | Disable automatic grapheme cluster segmentation (Mode 2027) |
| `NO_SHELL_INTEGRATION` | Disable OSC 133 shell integration sequences |
| `NO_CLIPBOARD` | Disable automatic clipboard writes via OSC 52 |

```java
EnumMap<ReadlineFlag, Integer> flags = new EnumMap<>(ReadlineFlag.class);
flags.put(ReadlineFlag.NO_PROMPT_REDRAW_ON_INTR, 1);

readline.readline(connection, prompt, handler, completions, 
        preProcessors, history, listener, flags);
```

## Completion

### Completion Class

Represents a single completion option:

```java
// Simple completion
Completion c1 = new Completion("value");

// With description
Completion c2 = new Completion("value", "Description shown in completion menu");

// With all options
Completion c3 = new Completion("value", "Description", true); // appendSpace after completion
```

### CompletionHandler Interface

Custom completion handling:

```java
public interface CompletionHandler {
    
    // Called when user presses Tab
    void complete(
            AeshCompleteOperation completeOperation,
            List<Completion> completions
    );
}
```

### Implementing Custom Completions

```java
public class CommandCompleter implements CompletionHandler {
    
    private final Map<String, List<String>> commandOptions;
    
    @Override
    public void complete(
            AeshCompleteOperation operation,
            List<Completion> completions) {
        
        String buffer = operation.getBuffer();
        String[] parts = buffer.split("\\s+");
        
        if (parts.length == 1) {
            // Complete command names
            for (String command : commandOptions.keySet()) {
                if (command.startsWith(parts[0])) {
                    completions.add(new Completion(command));
                }
            }
        } else {
            // Complete command options
            String command = parts[0];
            String optionPrefix = parts[parts.length - 1];
            
            List<String> options = commandOptions.get(command);
            if (options != null) {
                for (String option : options) {
                    if (option.startsWith(optionPrefix)) {
                        completions.add(new Completion(option));
                    }
                }
            }
        }
    }
}
```

## History

### History Interface

```java
public interface History {
    
    // Add entry to history
    void add(String entry);
    
    // Get entry at index
    String get(int index);
    
    // Get current entry
    String getCurrent();
    
    // Get number of entries
    int size();
    
    // Navigate history
    String getPrevious();
    String getNext();
    
    // Search history
    String search(String pattern);
    String searchReverse(String pattern);
    
    // Clear history
    void clear();
    
    // Get all entries
    List<String> getAll();
}
```

### InMemoryHistory

Non-persistent history stored in memory:

```java
History history = new InMemoryHistory(500); // Max 500 entries
```

### FileHistory

Persistent history saved to file:

```java
History history = new FileHistory(
        new File(".history"),  // File path
        500,                   // Max entries
        true                   // Append new entries
);
```

### Custom History Example

```java
public class FilteredHistory implements History {
    
    private final History delegate;
    private final Predicate<String> filter;
    
    public FilteredHistory(History delegate, Predicate<String> filter) {
        this.delegate = delegate;
        this.filter = filter;
    }
    
    @Override
    public void add(String entry) {
        // Only add entries that pass the filter
        if (filter.test(entry)) {
            delegate.add(entry);
        }
    }
    
    // Delegate other methods...
}

// Usage: Don't store commands starting with space
History history = new FilteredHistory(
        new InMemoryHistory(500),
        entry -> !entry.startsWith(" ")
);
```

## EditMode

### EditMode.Mode

```java
public enum Mode {
    EMACS,  // Emacs-style editing (default)
    VI      // Vi-style editing
}
```

### EditModeBuilder

```java
EditMode editMode = EditModeBuilder.builder()
        .mode(EditMode.Mode.EMACS)
        .create();

Readline readline = ReadlineBuilder.builder()
        .editMode(editMode)
        .build();
```

### Key Bindings

Default Emacs key bindings:

| Key | Action |
|-----|--------|
| `Ctrl-A` | Move to beginning of line |
| `Ctrl-E` | Move to end of line |
| `Ctrl-B` | Move back one character |
| `Ctrl-F` | Move forward one character |
| `Alt-B` | Move back one word |
| `Alt-F` | Move forward one word |
| `Ctrl-D` | Delete character under cursor / EOF |
| `Ctrl-H` / `Backspace` | Delete character before cursor |
| `Ctrl-K` | Kill to end of line |
| `Ctrl-U` | Kill to beginning of line |
| `Ctrl-W` | Kill word before cursor |
| `Ctrl-Y` | Yank (paste) killed text |
| `Ctrl-L` | Clear screen |
| `Ctrl-R` | Reverse history search |
| `Ctrl-P` / `Up` | Previous history entry |
| `Ctrl-N` / `Down` | Next history entry |
| `Tab` | Complete |
| `Ctrl-C` | Interrupt |

Default Vi key bindings (insert mode):

| Key | Action |
|-----|--------|
| `Escape` | Enter command mode |
| `Ctrl-H` | Delete character before cursor |

Vi command mode:

| Key | Action |
|-----|--------|
| `h`, `l` | Move left/right |
| `w`, `b` | Move by word |
| `0`, `$` | Beginning/end of line |
| `x` | Delete character |
| `dd` | Delete line |
| `i`, `a` | Enter insert mode |
| `k`, `j` | Previous/next history |
| `/` | Search history |

## Thread Safety

The `Readline` class is thread-safe. However:

- Only one `readline()` call can be active at a time
- Calling `readline()` while already reading throws `IllegalStateException`
- The callback is invoked on the same thread that handles terminal input

```java
// This will throw IllegalStateException
readline.readline(connection, "$ ", input1 -> {
    readline.readline(connection, "nested> ", input2 -> {
        // WRONG: Can't nest readline calls
    });
});

// Correct pattern: call readline again after handling input
readline.readline(connection, "$ ", input -> {
    processInput(input);
    // Schedule next read
    readline.readline(connection, "$ ", this);
});
```

## Complete Example

```java
import org.aesh.readline.*;
import org.aesh.readline.history.*;
import org.aesh.readline.editing.*;
import org.aesh.terminal.Connection;
import org.aesh.terminal.tty.Size;

import java.io.File;
import java.util.*;
import java.util.function.*;

public class InteractiveShell {
    
    private final Readline readline;
    private final Map<String, Consumer<String>> commands;
    private boolean running = true;
    
    public InteractiveShell() {
        // Configure readline
        this.readline = ReadlineBuilder.builder()
                .editMode(EditModeBuilder.builder()
                        .mode(EditMode.Mode.EMACS)
                        .create())
                .history(new FileHistory(new File(".myshell_history"), 1000, true))
                .enableHistory(true)
                .historySize(1000)
                .build();
        
        // Register commands
        this.commands = new HashMap<>();
        commands.put("help", this::showHelp);
        commands.put("clear", this::clearScreen);
        commands.put("history", this::showHistory);
        commands.put("exit", arg -> running = false);
    }
    
    public void start(Connection connection) {
        // Welcome message
        connection.write("Welcome to MyShell! Type 'help' for commands.\n");
        
        // Start read loop
        read(connection);
    }
    
    private void read(Connection connection) {
        if (!running) {
            connection.write("Goodbye!\n");
            connection.close();
            return;
        }
        
        List<Completion> completions = getCompletions();
        
        readline.readline(connection, createPrompt(), completions, input -> {
            if (input == null) {
                // EOF (Ctrl-D)
                running = false;
            } else if (!input.trim().isEmpty()) {
                processCommand(connection, input.trim());
            }
            
            // Continue reading
            read(connection);
        });
    }
    
    private Prompt createPrompt() {
        return new Prompt(new TerminalString(
                "myshell> ",
                new TerminalColor(Color.GREEN, Color.DEFAULT),
                CharacterType.BOLD
        ));
    }
    
    private List<Completion> getCompletions() {
        List<Completion> completions = new ArrayList<>();
        for (String cmd : commands.keySet()) {
            completions.add(new Completion(cmd, "Execute " + cmd + " command"));
        }
        return completions;
    }
    
    private void processCommand(Connection connection, String input) {
        String[] parts = input.split("\\s+", 2);
        String command = parts[0];
        String args = parts.length > 1 ? parts[1] : "";
        
        Consumer<String> handler = commands.get(command);
        if (handler != null) {
            handler.accept(args);
        } else {
            connection.write("Unknown command: " + command + "\n");
        }
    }
    
    private void showHelp(String args) {
        // Implementation
    }
    
    private void clearScreen(String args) {
        // Implementation
    }
    
    private void showHistory(String args) {
        // Implementation
    }
    
    public static void main(String[] args) {
        // Create connection and start shell
        // See Terminal documentation for connection setup
    }
}
```

## Error Handling

### Handling EOF

```java
readline.readline(connection, "$ ", input -> {
    if (input == null) {
        // User pressed Ctrl-D (EOF)
        // Clean up and exit
        connection.close();
        return;
    }
    // Process input...
});
```

### Handling Interrupts

```java
// Handle Ctrl-C in the connection
connection.setSignalHandler(signal -> {
    if (signal == Signal.INT) {
        // Ctrl-C pressed
        // Cancel current operation
    }
});
```

### Exception Handling

```java
readline.readline(connection, "$ ", input -> {
    try {
        processInput(input);
    } catch (Exception e) {
        connection.write("Error: " + e.getMessage() + "\n");
    }
    
    // Continue reading even after errors
    readline.readline(connection, "$ ", this);
});
```

## Performance Considerations

1. **Reuse Readline instances** - Create once, use for all reads
2. **Limit history size** - Large histories consume memory
3. **Use file history for persistence** - But be aware of I/O overhead
4. **Pre-compute completions** - Don't calculate on every Tab press
5. **Handle long-running operations** - Don't block the input thread

```java
// Good: Reuse readline instance
private final Readline readline = ReadlineBuilder.builder().build();

public void readInput(Connection connection) {
    readline.readline(connection, "$ ", this::handleInput);
}

// Bad: Create new instance for each read
public void readInput(Connection connection) {
    Readline readline = ReadlineBuilder.builder().build(); // Wasteful!
    readline.readline(connection, "$ ", this::handleInput);
}
```
