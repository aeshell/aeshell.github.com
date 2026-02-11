---
date: '2026-02-11T14:00:00+01:00'
draft: false
title: 'Table Display'
weight: 19
---

The table display utility renders structured data as formatted text tables in the terminal. It supports multiple border styles, automatic column width calculation, numeric formatting, and multi-line headers.

## Quick Start

```java
import org.aesh.util.table.Table;
import org.aesh.util.table.TableStyle;

String output = Table.<User>builder()
        .maxWidth(80)
        .style(TableStyle.DUCKDB)
        .column("Name", u -> u.getName())
        .column("Email", u -> u.getEmail())
        .build()
        .render(userList);

invocation.println(output);
```

Output:

```
┌──────┬──────────────────┐
│ Name │      Email       │
├──────┼──────────────────┤
│ Alice│ alice@example.com│
│ Bob  │ bob@example.com  │
└──────┴──────────────────┘
```

## Static API

For one-off table rendering, use the static `render` methods directly:

```java
import org.aesh.util.table.Table;

import java.util.Arrays;
import java.util.List;
import java.util.function.Function;

List<String> headers = Arrays.asList("Name", "Age", "Score");
List<Function<User, Object>> accessors = Arrays.asList(
        u -> u.getName(),
        u -> u.getAge(),
        u -> u.getScore()
);

// Render with default DUCKDB style
String output = Table.render(80, users, headers, accessors);

// Render with a specific style
String output = Table.render(80, users, headers, accessors,
        TableStyle.SQLITE.characters());
```

## Builder API

The builder API pairs each header with its accessor function, preventing mismatched list sizes:

```java
Table<User> table = Table.<User>builder()
        .maxWidth(120)
        .style(TableStyle.DOUBLE)
        .column("Name", u -> u.getName())
        .column("Age", u -> u.getAge())
        .column("Score", u -> u.getScore())
        .build();

// The built Table instance is reusable
String output1 = table.render(activeUsers);
String output2 = table.render(inactiveUsers);
```

You can also pass custom character maps to the builder instead of a predefined style:

```java
Table.<User>builder()
        .characters(myCustomCharacterMap)
        .column("Name", u -> u.getName())
        .build();
```

## Table Styles

Four predefined styles are available via `TableStyle`:

### POSTGRES

Simple ASCII without outside borders, similar to PostgreSQL's `psql` output:

```java
Table.render(80, users, headers, accessors, TableStyle.POSTGRES.characters());
```

```
 Name  | Age | Score
-------+-----+------
 Alice |  30 | 95.50
 Bob   |  25 | 87.12
```

### SQLITE

ASCII with full outside borders:

```java
Table.render(80, users, headers, accessors, TableStyle.SQLITE.characters());
```

```
+-------+-----+-------+
| Name  | Age | Score |
+-------+-----+-------+
| Alice |  30 | 95.50 |
| Bob   |  25 | 87.12 |
+-------+-----+-------+
```

### DUCKDB

Unicode box-drawing characters (the default style):

```java
Table.render(80, users, headers, accessors, TableStyle.DUCKDB.characters());
```

```
┌───────┬─────┬───────┐
│ Name  │ Age │ Score │
├───────┼─────┼───────┤
│ Alice │  30 │ 95.50 │
│ Bob   │  25 │ 87.12 │
└───────┴─────┴───────┘
```

### DOUBLE

Double-line box-drawing with row separators between data rows:

```java
Table.render(80, users, headers, accessors, TableStyle.DOUBLE.characters());
```

```
╔═══════╦═════╦═══════╗
║ Name  ║ Age ║ Score ║
╠═══════╬═════╬═══════╣
║ Alice ║  30 ║ 95.50 ║
╟───────╫─────╫───────╣
║ Bob   ║  25 ║ 87.12 ║
╚═══════╩═════╩═══════╝
```

## Numeric Formatting

Column types are detected automatically from the data:

| Value Type | Alignment | Formatting |
|------------|-----------|------------|
| `String` | Left-aligned | As-is |
| `Integer`, `Long` | Right-aligned | `%d` format |
| `Float`, `Double` | Right-aligned | 2 decimal places |
| `null` | Left-aligned | Rendered as `"null"` |

```java
Table.<Item>builder()
        .column("Product", i -> i.name)        // left-aligned string
        .column("Quantity", i -> i.quantity)     // right-aligned integer
        .column("Price", i -> i.price)           // right-aligned, 2 decimals
        .build()
        .render(items);
```

## Multi-line Headers

Headers can span multiple lines using `System.lineSeparator()`:

```java
List<String> headers = Arrays.asList(
        "First" + System.lineSeparator() + "Name",
        "Email"
);
```

```
┌───────┬──────────────────┐
│ First │                  │
│ Name  │      Email       │
├───────┼──────────────────┤
│ Alice │ alice@example.com│
└───────┴──────────────────┘
```

Headers are center-aligned within their column, and shorter headers are padded with empty lines to match the tallest header.

## Custom Border Characters

### Using Templates

Define custom borders with a visual 5-character-wide template. The template is a multi-line string where each line represents a part of the table border:

```java
import org.aesh.util.table.TableCharacters;

// 6-line template: top border, header, header-body separator, body, row separator, bottom border
String template =
        "╔═╦═╗" + System.lineSeparator() +
        "║h║h║" + System.lineSeparator() +
        "╠═╬═╣" + System.lineSeparator() +
        "║v║v║" + System.lineSeparator() +
        "╟─╫─╢" + System.lineSeparator() +
        "╚═╩═╝";

Map<String, String> chars = TableCharacters.templateToMap(template,
        TableStyle.DUCKDB.characters());
```

Templates with 3, 5, or 6 lines are supported, providing increasing levels of border detail.

### Using Shorthand Expansion

Define minimal character sets and expand them to all positions:

```java
Map<String, String> shorthand = new HashMap<>();
shorthand.put(TableCharacters.VERTICAL, "│");
shorthand.put(TableCharacters.HORIZONTAL, "─");
shorthand.put(TableCharacters.INTERSECT, "┼");

// Expand to all border positions (second parameter enables outside borders)
Map<String, String> full = TableCharacters.convertToFullNames(shorthand, true);
```

## ANSI Color Support

Add ANSI color to table borders using `TableCharacters.prefix()`:

```java
import org.aesh.terminal.utils.ANSI;
import org.aesh.util.table.TableCharacters;

Map<String, String> colored = TableCharacters.prefix(
        ANSI.CYAN_TEXT, TableStyle.DUCKDB.characters());

String output = Table.render(80, users, headers, accessors, colored);
```

This wraps each border character with the ANSI color prefix and `ANSI.RESET` suffix, so data values remain unaffected.

## Using Tables in Commands

A complete command that renders tabular output:

```java
import org.aesh.command.Command;
import org.aesh.command.CommandDefinition;
import org.aesh.command.CommandResult;
import org.aesh.command.invocation.CommandInvocation;
import org.aesh.util.table.Table;
import org.aesh.util.table.TableStyle;

@CommandDefinition(name = "users", description = "List all users")
public class ListUsersCommand implements Command<CommandInvocation> {

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        List<User> users = loadUsers();

        String table = Table.<User>builder()
                .maxWidth(invocation.getShell().size().getWidth())
                .style(TableStyle.DUCKDB)
                .column("Name", u -> u.getName())
                .column("Email", u -> u.getEmail())
                .column("Role", u -> u.getRole())
                .build()
                .render(users);

        invocation.getShell().writeln(table);
        return CommandResult.SUCCESS;
    }
}
```

Using `invocation.getShell().size().getWidth()` adapts the table to the current terminal width.

## Validation Helpers

`TableCharacters` provides methods to inspect character maps:

```java
// Check if the map defines a complete outside border
boolean bordered = TableCharacters.hasOutsideBorder(characters);

// Check if the map defines row separators between data rows
boolean separated = TableCharacters.hasRowSeparator(characters);

// Check if the map has the minimum required entries for rendering
boolean valid = TableCharacters.isValid(characters);
```

If an invalid character map is passed to `Table.render()`, it falls back to the `SQLITE` style automatically.

## API Reference

### Table

| Method | Description |
|--------|-------------|
| `Table.render(maxWidth, values, headers, accessors)` | Render with default DUCKDB style |
| `Table.render(maxWidth, values, headers, accessors, characters)` | Render with custom border characters |
| `Table.builder()` | Create a builder for fluent configuration |
| `table.render(values)` | Render using a pre-built Table instance |

### Table.Builder

| Method | Description |
|--------|-------------|
| `maxWidth(int)` | Set maximum table width (default: 80) |
| `style(TableStyle)` | Set a predefined border style |
| `characters(Map)` | Set a custom character map |
| `column(header, accessor)` | Add a column with header and value accessor |
| `build()` | Build an immutable `Table` instance |

### TableCharacters

| Method | Description |
|--------|-------------|
| `convertToFullNames(map, border)` | Expand shorthand V/H/X to full position names |
| `templateToMap(template, defaultMap)` | Parse a visual template into a character map |
| `prefix(ansiPrefix, map)` | Wrap border characters with ANSI color codes |
| `hasOutsideBorder(map)` | Check for complete outside border definition |
| `hasRowSeparator(map)` | Check for row separator definition |
| `isValid(map)` | Check for minimum required entries |
| `lineSplit(headers)` | Split multi-line headers into aligned rows |

### TableStyle

| Style | Borders | Characters | Row Separators |
|-------|---------|------------|----------------|
| `POSTGRES` | None | ASCII (`\|`, `-`, `+`) | No |
| `SQLITE` | Full | ASCII (`\|`, `-`, `+`) | No |
| `DUCKDB` | Full | Unicode box-drawing | No |
| `DOUBLE` | Full | Double-line box-drawing | Yes |
