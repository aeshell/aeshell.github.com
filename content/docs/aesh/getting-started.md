---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Getting Started with Æsh'
weight: 2
---

This guide takes you from zero to a working CLI application.

## Installation

Add the following dependency to your Maven project:

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>aesh</artifactId>
  <version>3.6</version>
</dependency>
```

For Gradle:

```groovy
dependencies {
    implementation 'org.aesh:aesh:3.6'
}
```

## Complete Working Example

Here is a fully runnable CLI application. Create a single file `TodoShell.java`:

```java
import org.aesh.AeshConsoleRunner;
import org.aesh.command.Command;
import org.aesh.command.CommandDefinition;
import org.aesh.command.CommandResult;
import org.aesh.command.invocation.CommandInvocation;
import org.aesh.command.option.Argument;
import org.aesh.command.option.Option;

import java.util.ArrayList;
import java.util.List;

public class TodoShell {

    // Shared state between commands
    static final List<String> todos = new ArrayList<>();

    @CommandDefinition(name = "add", description = "Add a todo item")
    public static class AddCommand implements Command<CommandInvocation> {

        @Argument(description = "The todo item text", required = true)
        private String item;

        @Override
        public CommandResult execute(CommandInvocation invocation) {
            todos.add(item);
            invocation.println("Added: " + item);
            return CommandResult.SUCCESS;
        }
    }

    @CommandDefinition(name = "list", description = "List all todo items")
    public static class ListCommand implements Command<CommandInvocation> {

        @Option(shortName = 'n', hasValue = false, description = "Show line numbers")
        private boolean numbered;

        @Override
        public CommandResult execute(CommandInvocation invocation) {
            if (todos.isEmpty()) {
                invocation.println("No items yet. Use 'add' to create one.");
                return CommandResult.SUCCESS;
            }
            for (int i = 0; i < todos.size(); i++) {
                if (numbered) {
                    invocation.println((i + 1) + ". " + todos.get(i));
                } else {
                    invocation.println("- " + todos.get(i));
                }
            }
            return CommandResult.SUCCESS;
        }
    }

    @CommandDefinition(name = "done", description = "Remove a completed item")
    public static class DoneCommand implements Command<CommandInvocation> {

        @Argument(description = "Item number to remove", required = true)
        private int number;

        @Override
        public CommandResult execute(CommandInvocation invocation) {
            if (number < 1 || number > todos.size()) {
                invocation.println("Invalid item number. Use 'list -n' to see numbers.");
                return CommandResult.FAILURE;
            }
            String removed = todos.remove(number - 1);
            invocation.println("Done: " + removed);
            return CommandResult.SUCCESS;
        }
    }

    public static void main(String[] args) {
        AeshConsoleRunner.builder()
                .command(AddCommand.class)
                .command(ListCommand.class)
                .command(DoneCommand.class)
                .prompt("[todo]$ ")
                .addExitCommand()
                .start();
    }
}
```

### Running It

Compile and run with Maven (or just use your IDE):

```bash
mvn compile exec:java -Dexec.mainClass="TodoShell"
```

### What You'll See

```
[todo]$ add Buy groceries
Added: Buy groceries
[todo]$ add Write tests
Added: Write tests
[todo]$ list -n
1. Buy groceries
2. Write tests
[todo]$ done 1
Done: Buy groceries
[todo]$ list
- Write tests
[todo]$ exit
```

Notice you get **tab completion** for command names and option names for free -- try pressing Tab after typing `li` or `done -`.

## How It Works

The example above demonstrates the core concepts:

1. **`@CommandDefinition`** marks a class as a command, giving it a name and description
2. **`@Option`** defines named flags and parameters (`--numbered` / `-n`)
3. **`@Argument`** defines positional arguments (the text after the command name)
4. **`Command<CommandInvocation>`** is the interface every command implements
5. **`AeshConsoleRunner`** wires everything together and starts the interactive shell

Æsh handles parsing, validation, tab completion, and help text automatically.

## Your First Command

The simplest possible command:

```java
import org.aesh.command.Command;
import org.aesh.command.CommandDefinition;
import org.aesh.command.CommandResult;
import org.aesh.command.invocation.CommandInvocation;

@CommandDefinition(name = "hello", description = "Print a greeting")
public class HelloCommand implements Command<CommandInvocation> {

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Hello from Æsh!");
        return CommandResult.SUCCESS;
    }
}
```

Run it with:

```java
public class Main {
    public static void main(String[] args) {
        AeshConsoleRunner.builder()
                .command(HelloCommand.class)
                .prompt("[myshell]$ ")
                .addExitCommand()
                .start();
    }
}
```

## Adding Options

Add options to your command using the `@Option` annotation:

```java
@CommandDefinition(name = "greet", description = "Greet someone")
public class GreetCommand implements Command<CommandInvocation> {

    @Option(shortName = 'n', description = "Name to greet")
    private String name = "World";

    @Option(shortName = 'u', hasValue = false, description = "Uppercase output")
    private boolean uppercase;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        String greeting = "Hello, " + name + "!";
        invocation.println(uppercase ? greeting.toUpperCase() : greeting);
        return CommandResult.SUCCESS;
    }
}
```

Usage:
```
[myshell]$ greet
Hello, World!
[myshell]$ greet --name Alice
Hello, Alice!
[myshell]$ greet -n Alice -u
HELLO, ALICE!
```

## Auto-Generated Help

Add `generateHelp = true` to get a `--help` option for free:

```java
@CommandDefinition(name = "greet", description = "Greet someone", generateHelp = true)
```

```
[myshell]$ greet --help
Usage: greet [-hu] [-n <name>]
Greet someone
  -h, --help        Display this help and exit
  -n, --name=<name> Name to greet
  -u, --uppercase   Uppercase output
```

## Non-Interactive CLI Tools

Use `AeshRuntimeRunner` to build single-command tools that take arguments from the command line (like `grep` or `curl`):

```java
import org.aesh.AeshRuntimeRunner;

public class GreetCli {
    public static void main(String[] args) {
        AeshRuntimeRunner.builder()
                .command(GreetCommand.class)
                .args(args)
                .execute();
    }
}
```

```bash
$ java -jar greet.jar --name Alice
Hello, Alice!
```

See [Console and Runtime Runners](../runners) for more details.

## Maven POM

A minimal `pom.xml` to get started:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>my-cli</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.aesh</groupId>
            <artifactId>aesh</artifactId>
            <version>3.6</version>
        </dependency>
    </dependencies>
</project>
```

## Next Steps

Now that you have a working application:

- Learn about [Options](../options), [Arguments](../arguments), and [Option Lists](../option-lists) for richer command-line interfaces
- Add [Tab Completion](../completers) for custom values
- Add [Validation](../validators) to check input before execution
- Use [Group Commands](../group-commands) for nested sub-commands (like `git remote add`)
- Explore [Console and Runtime Runners](../runners) for different execution modes
- Browse complete working code in the [aesh-examples repository](https://github.com/aeshell/aesh-examples)
