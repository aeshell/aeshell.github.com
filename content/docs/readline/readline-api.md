---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Readline API'
weight: 3
---

The `Readline` class provides the main API for reading input from a terminal.

## ReadlineBuilder

Configure and create a `Readline` instance:

```java
Readline readline = ReadlineBuilder.builder()
        .editMode(EditModeBuilder.builder().create())
        .history(new FileHistory(new File(".history"), 100))
        .enableHistory(true)
        .historySize(50)
        .historyFile(".history")
        .completionHandler(new SimpleCompletionHandler())
        .build();
```

## Reading Input

### Basic Read

```java
readline.readline(connection, prompt, input -> {
    if (input != null) {
        // Handle input
    }
});
```

### Read with Completions

```java
List<Completion> completions = Arrays.asList(
    new Completion("option1", "Description 1"),
    new Completion("option2", "Description 2")
);

readline.readline(connection, prompt, completions, input -> {
    // Handle input
});
```

### Read with Pre-processors

```java
List<Function<String, Optional<String>>> preProcessors = Arrays.asList(
    input -> input.trim().isEmpty() ? Optional.of("default") : Optional.of(input)
);

readline.readline(connection, prompt, completions, preProcessors, input -> {
    // Handle input
});
```

### Read with Custom History

```java
History customHistory = new InMemoryHistory(200);

readline.readline(connection, prompt, completions, preProcessors, customHistory, input -> {
    // Handle input
});
```

### Full Signature

```java
public void readline(
    Connection conn,
    Prompt prompt,
    Consumer<String> requestHandler,
    List<Completion> completions,
    List<Function<String, Optional<String>>> preProcessors,
    History history,
    CursorListener listener,
    EnumMap<ReadlineFlag, Integer> flags
)
```

## Thread Safety

`Readline` is thread-safe. It will not accept new `readline()` calls while currently reading input. If you try to call `readline()` while already reading, an `IllegalStateException` will be thrown.

## ReadlineFlag

Flags to control readline behavior:

| Flag | Description |
|------|-------------|
| `NO_PROMPT_REDRAW_ON_INTR` | Don't redraw prompt on interrupt |

## Complete Signatures

```java
// Minimal
readline(Connection conn, String prompt, Consumer<String> requestHandler)

// With completions
readline(Connection conn, String prompt, Consumer<String> requestHandler, List<Completion> completions)

// With Prompt object
readline(Connection conn, Prompt prompt, Consumer<String> requestHandler)

// With Prompt and completions
readline(Connection conn, Prompt prompt, Consumer<String> requestHandler, List<Completion> completions)

// With pre-processors
readline(Connection conn, Prompt prompt, Consumer<String> requestHandler, 
          List<Completion> completions, List<Function<String, Optional<String>>> preProcessors)

// With custom history
readline(Connection conn, Prompt prompt, Consumer<String> requestHandler, 
          List<Completion> completions, List<Function<String, Optional<String>>> preProcessors, 
          History history)

// With cursor listener
readline(Connection conn, Prompt prompt, Consumer<String> requestHandler, 
          List<Completion> completions, List<Function<String, Optional<String>>> preProcessors, 
          History history, CursorListener listener)

// With flags
readline(Connection conn, Prompt prompt, Consumer<String> requestHandler, 
          List<Completion> completions, List<Function<String, Optional<String>>> preProcessors, 
          History history, CursorListener listener, EnumMap<ReadlineFlag, Integer> flags)
```
