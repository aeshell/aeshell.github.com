---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Getting Started with Æsh Readline'
weight: 2
---

This guide will help you get started with Æsh Readline quickly.

## Installation

Add the following dependency to your Maven project:

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>readline</artifactId>
  <version>3.6</version>
</dependency>
```

For Gradle:

```groovy
dependencies {
    implementation 'org.aesh:readline:3.6'
}
```

## Basic Example

A simple readline example:

```java
import org.aesh.readline.Readline;
import org.aesh.readline.ReadlineBuilder;
import org.aesh.tty.terminal.TerminalConnection;

public class SimpleExample {

    public static void main(String... args) {
        TerminalConnection connection = new TerminalConnection();
        Readline readline = ReadlineBuilder.builder().enableHistory(false).build();
        
        read(connection, readline, "[prompt]$ ");
        connection.openBlocking();
    }

    private static void read(TerminalConnection connection, Readline readline, String prompt) {
        readline.readline(connection, prompt, input -> {
            if (input != null && input.equals("exit")) {
                connection.write("Goodbye!\n");
                connection.close();
            }
            else {
                connection.write("You entered: " + input + "\n");
                read(connection, readline, prompt);
            }
        });
    }
}
```

## With History

Enable history for persistent command history:

```java
import org.aesh.readline.Readline;
import org.aesh.readline.ReadlineBuilder;
import org.aesh.readline.history.FileHistory;
import org.aesh.tty.terminal.TerminalConnection;
import java.io.File;

public class HistoryExample {

    public static void main(String... args) {
        TerminalConnection connection = new TerminalConnection();
        
        Readline readline = ReadlineBuilder.builder()
                .history(new FileHistory(new File(".history"), 100))
                .build();
        
        read(connection, readline, "[history]$ ");
        connection.openBlocking();
    }

    private static void read(TerminalConnection connection, Readline readline, String prompt) {
        readline.readline(connection, prompt, input -> {
            if (input != null && input.equals("exit")) {
                connection.close();
            }
            else {
                connection.write("Echo: " + input + "\n");
                read(connection, readline, prompt);
            }
        });
    }
}
```

## With Completion

Add tab completion for commands:

```java
import org.aesh.readline.Readline;
import org.aesh.readline.ReadlineBuilder;
import org.aesh.readline.completion.Completion;
import org.aesh.tty.terminal.TerminalConnection;
import java.util.Arrays;
import java.util.List;

public class CompletionExample {

    public static void main(String... args) {
        TerminalConnection connection = new TerminalConnection();
        
        List<Completion> completions = Arrays.asList(
                new Completion("hello", "Say hello"),
                new Completion("goodbye", "Say goodbye"),
                new Completion("exit", "Exit the program")
        );
        
        Readline readline = ReadlineBuilder.builder()
                .enableHistory(true)
                .build();
        
        read(connection, readline, "[prompt]$ ", completions);
        connection.openBlocking();
    }

    private static void read(TerminalConnection connection, Readline readline, 
                            String prompt, List<Completion> completions) {
        readline.readline(connection, prompt, input -> {
            if (input != null) {
                if (input.equals("exit")) {
                    connection.close();
                }
                else if (input.equals("hello")) {
                    connection.write("Hello!\n");
                }
                else if (input.equals("goodbye")) {
                    connection.write("Goodbye!\n");
                }
                read(connection, readline, prompt, completions);
            }
        }, completions);
    }
}
```

## Next Steps

Now that you understand the basics:

- Learn about the [Readline API](../readline-api) for advanced features
- Explore [Edit Modes](../edit-modes) for Emacs and Vi key bindings
- Discover [Completion](../completion) strategies for tab completion
- Check out [Remote Connectivity](../connectivity) for SSH, Telnet, and WebSocket support
- Browse complete working code in [Examples and Tutorials](../examples)

## Working Examples

The [aesh-examples repository](https://github.com/aeshell/aesh-examples) contains several readline examples:

- **[getting-started](https://github.com/aeshell/aesh-examples/tree/master/readline/getting-started)** - Simple readline example
- **[getting-started-ext](https://github.com/aeshell/aesh-examples/tree/master/readline/getting-started-ext)** - Extended example with more features
- **[shell](https://github.com/aeshell/aesh-examples/tree/master/readline/shell)** - Simple shell implementation
- **[snake](https://github.com/aeshell/aesh-examples/tree/master/readline/snake)** - Snake game demonstrating terminal control

See the [Examples and Tutorials](../examples) page for detailed information about all available examples.
