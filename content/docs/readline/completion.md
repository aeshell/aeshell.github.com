---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Completion'
---

Æsh Readline supports tab completion for commands and options.

## Completion Class

Define a completion option:

```java
import org.aesh.readline.completion.Completion;

Completion completion = new Completion("option", "Description");
```

### Constructors

```java
// Name only
Completion c1 = new Completion("option");

// Name with description
Completion c2 = new Completion("option", "Description");

// Name with description and terminal style
Completion c3 = new Completion(
    "option", 
    "Description", 
    TerminalColor.GREEN
);
```

## Basic Completion

```java
import org.aesh.readline.Readline;
import org.aesh.readline.ReadlineBuilder;
import org.aesh.readline.completion.Completion;
import org.aesh.tty.terminal.TerminalConnection;
import java.util.Arrays;
import java.util.List;

public class BasicCompletion {

    public static void main(String... args) {
        TerminalConnection connection = new TerminalConnection();
        
        List<Completion> completions = Arrays.asList(
            new Completion("hello", "Say hello"),
            new Completion("goodbye", "Say goodbye"),
            new Completion("exit", "Exit program"),
            new Completion("help", "Show help")
        );
        
        Readline readline = ReadlineBuilder.builder()
                .enableHistory(false)
                .build();
        
        read(connection, readline, "[prompt]$ ", completions);
        connection.openBlocking();
    }

    private static void read(TerminalConnection connection, Readline readline, 
                            String prompt, List<Completion> completions) {
        readline.readline(connection, prompt, completions, input -> {
            if (input != null) {
                if (input.equals("exit")) {
                    connection.close();
                }
                else {
                    connection.write("You entered: " + input + "\n");
                    read(connection, readline, prompt, completions);
                }
            }
        });
    }
}
```

## Dynamic Completion

Update completions based on context:

```java
List<Completion> completions = new ArrayList<>();

readline.readline(connection, prompt, completions, input -> {
    if (input != null) {
        // Update completions based on context
        completions.clear();
        if (someCondition) {
            completions.add(new Completion("option1"));
        }
        // ...
        read(connection, readline, prompt, completions);
    }
});
```

## CompletionHandler

Custom completion handler:

```java
import org.aesh.readline.completion.CompletionHandler;
import org.aesh.readline.completion.SimpleCompletionHandler;

CompletionHandler handler = new SimpleCompletionHandler();

// Or implement custom handler
CompletionHandler customHandler = new CompletionHandler() {
    @Override
    public void addCompletions(List<Completion> completions) {
        // Add completions
    }
    
    @Override
    public void clear() {
        // Clear completions
    }
    
    @Override
    public List<Completion> getCompletions() {
        // Get current completions
        return Collections.emptyList();
    }
};

Readline readline = ReadlineBuilder.builder()
        .completionHandler(customHandler)
        .build();
```

## Partial Matching

Completions automatically filter based on partial input:

```java
List<Completion> completions = Arrays.asList(
    new Completion("git-add", "Add files"),
    new Completion("git-commit", "Commit changes"),
    new Completion("git-push", "Push to remote"),
    new Completion("git-pull", "Pull from remote")
);
```

User types `git-` and presses Tab - shows all four options.
User types `git-p` and presses Tab - shows `git-push` and `git-pull`.

## Completion Groups

Organize completions logically:

```java
List<Completion> fileCompletions = Arrays.asList(
    new Completion("read", "Read file"),
    new Completion("write", "Write file"),
    new Completion("delete", "Delete file")
);

List<Completion> networkCompletions = Arrays.asList(
    new Completion("download", "Download from URL"),
    new Completion("upload", "Upload to URL")
);

// Combine based on context
```

## Terminal Styled Completions

```java
import org.aesh.readline.terminal.formatting.TerminalColor;

Completion colored = new Completion(
    "important-option",
    "Important option",
    TerminalColor.RED
);
```
