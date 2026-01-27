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

## Custom Color Completer

```java
public class ColorCompleter implements OptionCompleter<CompleterInvocation> {

    private static final List<String> COLORS = Arrays.asList(
            "red", "green", "blue", "yellow", "orange", "purple",
            "cyan", "magenta", "white", "black", "gray", "pink"
    );

    @Override
    public void complete(CompleterInvocation invocation) {
        String input = invocation.getGivenCompleteValue();
        
        if (input == null || input.isEmpty()) {
            // No input yet, show all colors
            invocation.addAllCompleterValues(COLORS);
        } else {
            // Filter colors that start with the input
            String lowerInput = input.toLowerCase();
            for (String color : COLORS) {
                if (color.startsWith(lowerInput)) {
                    invocation.addCompleterValue(color);
                }
            }
        }
    }
}
```

Usage:

```java
@CommandDefinition(name = "theme", description = "Set application theme")
public class ThemeCommand implements Command<CommandInvocation> {

    @Option(
        shortName = 'b',
        name = "background",
        completer = ColorCompleter.class,
        description = "Background color"
    )
    private String backgroundColor;

    @Option(
        shortName = 'f',
        name = "foreground",
        completer = ColorCompleter.class,
        description = "Foreground/text color"
    )
    private String foregroundColor;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Setting theme: " + foregroundColor + " on " + backgroundColor);
        return CommandResult.SUCCESS;
    }
}
```

When users press Tab:
```
[myapp]$ theme --background <TAB>
red     green   blue    yellow  orange  purple
cyan    magenta white   black   gray    pink

[myapp]$ theme --background gr<TAB>
gray    green

[myapp]$ theme --background green --foreground wh<TAB>
white
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
