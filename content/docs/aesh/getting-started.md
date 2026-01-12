---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Getting Started with Æsh'
weight: 2
---

This guide will help you get started with Æsh quickly.

## Installation

Add the following dependency to your Maven project:

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>aesh</artifactId>
  <version>1.7</version>
</dependency>
```

For Gradle:

```groovy
dependencies {
    implementation group: 'org.aesh', name: 'aesh', version: '1.7'
}
```

## Your First Command

The simplest way to create a command is to annotate a class that implements the `Command` interface:

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

## Running your first Command

Use the `AeshConsoleRunner` to start an interactive console:

```java
import org.aesh.AeshConsoleRunner;

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

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Hello, " + name + "!");
        return CommandResult.SUCCESS;
    }
}
```

Now users can run `greet` (defaults to "World") or `greet --name Alice` / `greet -n Alice`.

## Multiple Commands

Register multiple commands by chaining the `command()` method:

```java
AeshConsoleRunner.builder()
        .command(HelloCommand.class)
        .command(GreetCommand.class)
        .prompt("[myshell]$ ")
        .addExitCommand()
        .start();
```
