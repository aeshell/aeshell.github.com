---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'History'
weight: 7
---

Æsh Readline provides flexible history management for command input.

## History Interface

```java
public interface History {
    void push(String line);
    String get(int index);
    List<String> getAll();
    int size();
    void clear();
    boolean isEnabled();
    void enable();
    void disable();
    String previous();
    String next();
    String first();
    String last();
    int getCurrentIndex();
    void setCurrentIndex(int index);
}
```

## InMemoryHistory

History stored in memory:

```java
import org.aesh.readline.history.InMemoryHistory;

History history = new InMemoryHistory(100);  // Max 100 entries
history.push("command1");
history.push("command2");

String previous = history.previous();
String next = history.next();
```

## FileHistory

History persisted to file:

```java
import org.aesh.readline.history.FileHistory;
import java.io.File;

File historyFile = new File(".aesh_history");
History history = new FileHistory(historyFile, 500);  // Max 500 entries
history.push("command1");  // Automatically saved to file
```

## Using History with ReadlineBuilder

```java
import org.aesh.readline.ReadlineBuilder;

Readline readline = ReadlineBuilder.builder()
        .history(new FileHistory(new File(".history"), 100))
        .enableHistory(true)
        .build();
```

## Programmatic History

```java
History history = ...;

// Add to history
history.push("git status");
history.push("git add .");
history.push("git commit -m 'fix'");

// Navigate history
String cmd1 = history.previous();  // "git commit -m 'fix'"
String cmd2 = history.previous();  // "git add ."
String cmd3 = history.next();      // "git commit -m 'fix'"

// Get all
List<String> allCommands = history.getAll();

// Get by index
String specificCommand = history.get(2);

// Get current position
int currentIndex = history.getCurrentIndex();

// Clear
history.clear();
```

## Enable/Disable History

```java
History history = ...;

history.enable();   // Enable history
history.disable();  // Disable history

boolean enabled = history.isEnabled();
```

## History Search

### Fuzzy Search (Ctrl+R)

Press `Ctrl+R` to open an interactive fuzzy history search inspired by [fzf](https://github.com/junegunn/fzf). Type to filter results, use `↑`/`↓` to navigate, and `Enter` to select. See [Fuzzy History Search]({{< ref "fuzzy-history-search" >}}) for full documentation.

### Traditional Search (Ctrl+S)

The traditional incremental forward search is available via `Ctrl+S`. The old `Ctrl+R` reverse incremental search can be restored by rebinding the key -- see [Fuzzy History Search]({{< ref "fuzzy-history-search" >}}#reverting-to-the-old-search) for details.

### Programmatic Search

Access history for search operations:

```java
History history = ...;

List<String> allHistory = history.getAll();
for (int i = 0; i < allHistory.size(); i++) {
    String cmd = allHistory.get(i);
    if (cmd.startsWith("git")) {
        // Found a git command
        String found = history.get(i);
    }
}
```

## History Persistence

`FileHistory` automatically persists to file:

```java
File historyFile = new File(".cmdline_history");
History history = new FileHistory(historyFile, 1000);

// These commands are automatically persisted
history.push("command1");
history.push("command2");

// History is loaded from file on construction
```

## History Size Limits

```java
// In-memory with limit
History memoryHistory = new InMemoryHistory(50);

// File-based with limit
History fileHistory = new FileHistory(new File(".history"), 200);
```

When the limit is reached, oldest entries are removed.

## Empty Input

Empty input is not added to history by default:

```java
history.push("");      // Not added (empty)
history.push("   ");   // Not added (whitespace only)
history.push("cmd");   // Added
```
