---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Completers'
weight: 8
---

Completers provide tab-completion suggestions for options and arguments.

## OptionCompleter Interface

```java
public interface OptionCompleter<T extends CompleterInvocation> {
    void complete(T completerInvocation);
}
```

## Built-in Completers

Æsh includes built-in completers:
- `BooleanOptionCompleter` - Completes boolean values
- `DefaultValueOptionCompleter` - Uses defined default values
- `FileOptionCompleter` - Completes file paths

## Custom File Completer

```java
public class JavaFileCompleter implements OptionCompleter<CompleterInvocation> {

    @Override
    public void complete(CompleterInvocation invocation) {
        String base = invocation.getGivenCompleteValue();
        File dir = base == null ? new File(".") : new File(base).getParentFile();
        String prefix = base == null ? "" : new File(base).getName();

        File[] files = dir.listFiles((d, name) -> 
            name.startsWith(prefix) && name.endsWith(".java")
        );

        if (files != null) {
            for (File f : files) {
                invocation.addCompleterValue(f.getName());
            }
        }
    }
}
```

Usage:

```java
@CommandDefinition(name = "compile")
public class CompileCommand implements Command<CommandInvocation> {

    @Option(
        name = "file",
        completer = JavaFileCompleter.class,
        description = "Java file to compile"
    )
    private String file;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Compiling: " + file);
        return CommandResult.SUCCESS;
    }
}
```

## CompleterInvocation

Provides access to:
- `getGivenCompleteValue()` - Current partial input
- `getAllGivenValues()` - All values for this option
- `getCommand()` - The command instance
- `getAeshContext()` - Access to runtime context
- `addCompleterValue(String)` - Add a completion suggestion

## Dynamic Completion Based on Other Options

```java
public class DatabaseTableCompleter implements OptionCompleter<CompleterInvocation> {

    @Override
    public void complete(CompleterInvocation invocation) {
        // Access the command to get other option values
        MyCommand cmd = (MyCommand) invocation.getCommand();
        
        if (cmd.database != null) {
            // Complete tables based on selected database
            invocation.addAllCompleterValues(
                getTablesForDatabase(cmd.database)
            );
        }
    }
}

@CommandDefinition(name = "query")
public class MyCommand implements Command<CommandInvocation> {

    @Option(name = "database", description = "Database name")
    private String database;

    @Option(
        name = "table",
        completer = DatabaseTableCompleter.class,
        description = "Table name"
    )
    private String table;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        return CommandResult.SUCCESS;
    }
}
```

## Completer for Arguments

Works with `@Arguments` as well:

```java
@Arguments(completer = CommandNameCompleter.class)
private List<String> commandNames;
```
