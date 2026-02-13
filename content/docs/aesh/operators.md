---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Operators'
weight: 16
---

Æsh supports Unix-style command operators for chaining commands, piping output, and redirecting I/O. Operators are enabled by default and can be configured through [Settings](/docs/aesh/settings).

## Operator Reference

### Pipeline

| Operator | Symbol | Description |
|----------|--------|-------------|
| Pipe | `\|` | Pass stdout of one command to stdin of the next |

### Redirection

| Operator | Symbol | Description |
|----------|--------|-------------|
| Redirect Out | `>` | Write stdout to a file (overwrites) |
| Append Out | `>>` | Append stdout to a file |
| Redirect In | `<` | Read stdin from a file |
| Redirect Error | `2>` | Write stderr to a file (overwrites) |
| Append Error | `2>>` | Append stderr to a file |
| Redirect All | `2>&1` | Merge stderr into stdout |

### Conditional and Sequencing

| Operator | Symbol | Description |
|----------|--------|-------------|
| AND | `&&` | Execute next command only if current succeeds |
| OR | `\|\|` | Execute next command only if current fails |
| Sequence | `;` | Execute next command unconditionally |

### Background

| Operator | Symbol | Description |
|----------|--------|-------------|
| Background | `&` | Run command in the background |

## Pipe

The pipe operator passes the output of one command as input to the next. Commands in a pipeline run sequentially, with stdout flowing from left to right.

```bash
# Filter output
ls | grep ".java"

# Chain multiple commands
cat data.csv | filter --column status --value active | sort --by name
```

### Writing Pipeable Commands

Commands that produce output for piping should write to the shell normally. Commands that consume piped input should read from `getInputLine()`:

```java
@CommandDefinition(name = "upper", description = "Convert input to uppercase")
public class UpperCommand implements Command<CommandInvocation> {

    @Override
    public CommandResult execute(CommandInvocation invocation)
            throws InterruptedException {
        String line;
        while ((line = invocation.getInputLine()) != null) {
            invocation.println(line.toUpperCase());
        }
        return CommandResult.SUCCESS;
    }
}
```

```bash
$ echo "hello world" | upper
HELLO WORLD
```

### Detecting Piped Context

Commands can adapt their output depending on whether they are being piped or writing directly to the terminal:

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    Operator operator = invocation.getOperator();

    if (operator == Operator.PIPE) {
        // Output is piped -- emit raw data, no formatting or colors
        invocation.println(data);
    } else {
        // Output goes to terminal -- use formatting
        invocation.println(formatTable(data));
    }

    return CommandResult.SUCCESS;
}
```

This is useful for commands like `ls` that show formatted tables on the terminal but emit plain filenames when piped.

## Output Redirection

### Overwrite (`>`)

Writes command output to a file, creating the file if it does not exist or overwriting it if it does. ANSI escape codes are stripped from the output.

```bash
# Save listing to file
ls > files.txt

# Save filtered results
ls | grep ".java" > java-files.txt
```

### Append (`>>`)

Appends command output to a file, creating it if it does not exist.

```bash
# Append to a log
echo "Build started" >> build.log
process --input data.csv >> build.log
echo "Build finished" >> build.log
```

### Error Redirection (`2>`, `2>>`)

Redirects stderr separately from stdout.

```bash
# Save errors to a file
compile src/ 2> errors.log

# Append errors
compile src/ 2>> errors.log
```

### Merge Stderr into Stdout (`2>&1`)

Combines stderr and stdout into a single stream, useful when piping or redirecting all output.

```bash
# Capture both stdout and stderr in one file
build 2>&1 > all-output.log
```

## Input Redirection

Reads command input from a file instead of the terminal.

```bash
# Process a file as input
process < input.txt

# Combine with output redirection
transform < input.csv > output.csv
```

Commands receiving redirected input read it through `getInputLine()`, the same way they receive piped input.

## Conditional Execution

### AND (`&&`)

Executes the next command only if the previous command returned `CommandResult.SUCCESS`.

```bash
# Only test if build succeeds
build && test

# Chain multiple dependent steps
build && test && package && deploy
```

If any command in the chain fails, the remaining commands are skipped.

### OR (`||`)

Executes the next command only if the previous command returned `CommandResult.FAILURE`.

```bash
# Fallback on failure
restore --from backup.db || restore --from backup-old.db

# Error handling
deploy || notify --message "Deploy failed"
```

### Sequence (`;`)

Executes the next command regardless of whether the previous command succeeded or failed.

```bash
# Always clean up
process ; cleanup

# Run independent commands
build ; test ; report
```

## Combining Operators

Operators can be combined in a single command line:

```bash
# Pipeline with conditional and redirection
build && test | grep FAIL > failures.txt

# Sequence with redirection
echo "Starting" >> log.txt ; process >> log.txt ; echo "Done" >> log.txt

# Conditional with fallback
deploy && echo "Success" || echo "Failed"
```

## File Path Resolution

Redirect operators resolve relative file paths against the current working directory. Absolute paths are used as-is.

```bash
# Relative path -- resolved against cwd
ls > output.txt

# Absolute path
ls > /tmp/output.txt
```

## Configuration

Operators are enabled by default. To disable them:

```java
Settings settings = SettingsBuilder.builder()
        .enableOperatorParser(false)   // Disable all operator parsing
        .build();
```

To selectively enable specific operator types:

```java
Settings settings = SettingsBuilder.builder()
        .enableOperatorParser(true)    // Enable operator parsing
        .setPipe(true)                 // Enable pipe operator
        .setRedirection(true)          // Enable redirection operators
        .build();
```

## Quoting and Escaping

Operators inside quotes are treated as literal characters, not as operators:

```bash
# The | here is literal, not a pipe
echo "value1 | value2"

# The > here is literal
echo "a > b"
```

Backslash escaping also prevents operator interpretation.

## Implementing Operator-Aware Commands

A complete example of a command designed to work well in pipelines:

```java
@CommandDefinition(name = "filter", description = "Filter lines by pattern")
public class FilterCommand implements Command<CommandInvocation> {

    @Option(shortName = 'p', required = true, description = "Pattern to match")
    private String pattern;

    @Option(shortName = 'v', hasValue = false,
            description = "Invert match (show non-matching lines)")
    private boolean invert;

    @Override
    public CommandResult execute(CommandInvocation invocation)
            throws InterruptedException {
        String line;
        boolean matched = false;

        while ((line = invocation.getInputLine()) != null) {
            boolean contains = line.contains(pattern);
            if (contains != invert) {
                invocation.println(line);
                matched = true;
            }
        }

        return matched ? CommandResult.SUCCESS : CommandResult.FAILURE;
    }
}
```

This command:
- Reads piped input or redirected input via `getInputLine()`
- Writes matching lines to stdout (available for further piping or redirection)
- Returns `FAILURE` if nothing matched, enabling conditional chaining:

```bash
# Only notify if errors were found
build 2>&1 | filter -p ERROR && notify --message "Build errors detected"
```
