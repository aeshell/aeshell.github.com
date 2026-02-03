---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Options'
weight: 4
---

The `@Option` annotation defines command-line options (flags with values).

## Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `name` | `String` | `""` | Option name (variable name if empty) |
| `shortName` | `char` | `'\u0000'` | Short name (e.g., `-v`) |
| `description` | `String` | `""` | Help description |
| `argument` | `String` | `""` | Value type description |
| `required` | `boolean` | `false` | Is option required? |
| `hasValue` | `boolean` | `true` | Does option accept a value? |
| `defaultValue` | `String[]` | `{}` | Default values |
| `askIfNotSet` | `boolean` | `false` | Prompt user if not set |
| `overrideRequired` | `boolean` | `false` | Override required validation |
| `acceptNameWithoutDashes` | `boolean` | `false` | Allow option name without `--` prefix |
| `negatable` | `boolean` | `false` | Enable `--no-{name}` form for boolean options |
| `negationPrefix` | `String` | `"no-"` | Prefix for negated form (e.g., `"without-"`) |
| `converter` | `Class<? extends Converter>` | `NullConverter.class` | Custom value converter |
| `completer` | `Class<? extends OptionCompleter>` | `NullOptionCompleter.class` | Custom completer |
| `validator` | `Class<? extends OptionValidator>` | `NullValidator.class` | Custom validator |
| `activator` | `Class<? extends OptionActivator>` | `NullActivator.class` | Custom activator |
| `renderer` | `Class<? extends OptionRenderer>` | `NullOptionRenderer.class` | Custom renderer |
| `parser` | `Class<? extends OptionParser>` | `AeshOptionParser.class` | Custom parser |

## Basic Example

```java
@CommandDefinition(name = "greet")
public class GreetCommand implements Command<CommandInvocation> {

    @Option(shortName = 'n', description = "Name to greet")
    private String name;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Hello, " + name + "!");
        return CommandResult.SUCCESS;
    }
}
```

Usage: `greet --name Alice` or `greet -n Alice`

## Boolean Flags

For boolean fields, `hasValue` can be `false`:

```java
@Option(shortName = 'v', hasValue = false, description = "Verbose output")
private boolean verbose;
```

Usage: `greet -v` or `greet --verbose`

## Negatable Options

Negatable options allow boolean flags to be explicitly set to `false` using a `--no-{name}` syntax. This is useful when you have a default value of `true` and want users to be able to disable the feature.

### Basic Usage

```java
@Option(hasValue = false, negatable = true, description = "Enable verbose output")
private boolean verbose;
```

Usage:
- `mycommand --verbose` sets `verbose` to `true`
- `mycommand --no-verbose` sets `verbose` to `false`

### Custom Negation Prefix

By default, the negation prefix is `"no-"`. You can customize it with the `negationPrefix` property:

```java
@Option(hasValue = false, negatable = true, negationPrefix = "without-",
        description = "Enable color output")
private boolean color;

@Option(hasValue = false, negatable = true, negationPrefix = "disable-",
        description = "Enable caching")
private boolean cache;
```

Usage:
- `mycommand --color` / `mycommand --without-color`
- `mycommand --cache` / `mycommand --disable-cache`

### Help Output

Negatable options automatically show both forms in help output:

```
Options:
  --verbose, --no-verbose       Enable verbose output
  --color, --without-color      Enable color output
```

### Completion Support

Tab completion works for both the positive and negated forms. Typing `--no-` and pressing Tab will show all available negated options.

### Complete Example

```java
@CommandDefinition(name = "build", description = "Build the project")
public class BuildCommand implements Command<CommandInvocation> {

    @Option(hasValue = false, negatable = true, defaultValue = "true",
            description = "Run tests during build")
    private boolean tests;

    @Option(hasValue = false, negatable = true, defaultValue = "true",
            description = "Enable compiler optimizations")
    private boolean optimize;

    @Option(hasValue = false, negatable = true, negationPrefix = "skip-",
            description = "Generate documentation")
    private boolean docs;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Building with:");
        invocation.println("  Tests: " + tests);
        invocation.println("  Optimize: " + optimize);
        invocation.println("  Docs: " + docs);
        return CommandResult.SUCCESS;
    }
}
```

Usage:
```bash
# Run with defaults (tests=true, optimize=true, docs=false)
$ build
Building with:
  Tests: true
  Optimize: true
  Docs: false

# Disable tests and optimization, enable docs
$ build --no-tests --no-optimize --docs
Building with:
  Tests: false
  Optimize: false
  Docs: true

# Use custom prefix
$ build --skip-docs
Building with:
  Tests: true
  Optimize: true
  Docs: false
```

### Important Notes

1. **Boolean types only** - The `negatable` property is only valid for `boolean` or `Boolean` field types. Using it with other types will result in a parsing exception.

2. **Combine with `hasValue = false`** - Negatable options should have `hasValue = false` since they are boolean flags.

3. **Default values** - Use `defaultValue = "true"` if you want the option enabled by default and allow users to disable it with the negated form.

## Required Options

```java
@Option(required = true, description = "Required file path")
private String filePath;
```

## Default Values

### Static Default Values

```java
@Option(defaultValue = "INFO", description = "Log level")
private String logLevel;
```

### Environment Variables

Use the `${ }` syntax to reference environment variables as default values:

```java
@Option(defaultValue = "${HOME}", description = "Home directory")
private String homeDir;

@Option(defaultValue = "${USER}", description = "Username")
private String username;

@Option(defaultValue = "${DATABASE_URL}", description = "Database connection URL")
private String databaseUrl;
```

If the environment variable is not set, the option will have no default value (null).

### Environment Variables with Fallback

Provide a fallback value after a colon (`:`) in case the environment variable is not set:

```java
@Option(defaultValue = "${PORT:8080}", description = "Server port")
private int port;

@Option(defaultValue = "${LOG_LEVEL:INFO}", description = "Log level")
private String logLevel;

@Option(defaultValue = "${DATABASE_HOST:localhost}", description = "Database host")
private String dbHost;
```

If `PORT` is not set, the default will be `8080`. If `LOG_LEVEL` is not set, the default will be `INFO`.

### System Properties

Reference Java system properties using the same syntax:

```java
@Option(defaultValue = "${user.home}", description = "User home directory")
private String userHome;

@Option(defaultValue = "${user.name}", description = "Current user")
private String userName;

@Option(defaultValue = "${java.io.tmpdir}", description = "Temp directory")
private String tempDir;

@Option(defaultValue = "${os.name}", description = "Operating system")
private String osName;
```

### System Properties with Fallback

```java
@Option(defaultValue = "${my.app.config:/etc/myapp/config}", description = "Config path")
private String configPath;
```

### Combined Example

```java
@CommandDefinition(name = "connect", description = "Connect to database")
public class ConnectCommand implements Command<CommandInvocation> {

    @Option(
        shortName = 'h',
        defaultValue = "${DB_HOST:localhost}",
        description = "Database host"
    )
    private String host;

    @Option(
        shortName = 'p',
        defaultValue = "${DB_PORT:5432}",
        description = "Database port"
    )
    private int port;

    @Option(
        shortName = 'u',
        defaultValue = "${DB_USER:${user.name}}",
        description = "Database user (defaults to system user)"
    )
    private String user;

    @Option(
        shortName = 'd',
        defaultValue = "${DB_NAME:myapp}",
        description = "Database name"
    )
    private String database;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println(String.format(
            "Connecting to %s@%s:%d/%s", user, host, port, database));
        return CommandResult.SUCCESS;
    }
}
```

Usage:
```bash
# Uses all defaults (environment variables or fallback values)
$ connect
Connecting to john@localhost:5432/myapp

# Override with environment variables
$ export DB_HOST=prod-db.example.com
$ export DB_PORT=5433
$ connect
Connecting to john@prod-db.example.com:5433/myapp

# Override with command-line options (highest priority)
$ connect -h dev-db.local -p 5434
Connecting to john@dev-db.local:5434/myapp
```

### Priority Order

When determining option values, Æsh uses this priority (highest to lowest):

1. **Command-line argument** - Explicitly provided by user
2. **Environment variable / System property** - If referenced in `defaultValue`
3. **Fallback value** - Value after `:` in the `defaultValue` expression
4. **Field initializer** - Java field initialization value
5. **null** - If nothing else is set

### Multiple Default Values

For options that accept multiple values (lists/arrays), provide multiple defaults:

```java
@Option(defaultValue = {"${DEFAULT_TAG:latest}", "stable"}, description = "Image tags")
private List<String> tags;
```

### Environment Variables in Arguments

The same syntax works with `@Argument`:

```java
@Argument(defaultValue = "${PWD:${user.dir}}", description = "Working directory")
private String workingDirectory;
```

### Best Practices

1. **Always provide fallbacks** - Use `${VAR:fallback}` to ensure sensible defaults when environment variables aren't set.

2. **Document expected variables** - Tell users which environment variables your application reads.

3. **Use consistent naming** - Follow conventions like `MYAPP_DATABASE_HOST` for your app's variables.

4. **Prefer environment variables for secrets** - Don't hardcode passwords; use `${DB_PASSWORD}` without a fallback.

```java
// Good: Password from environment, no fallback (will be null if not set)
@Option(defaultValue = "${DB_PASSWORD}", description = "Database password")
private String password;

// In execute(), check if password was provided
if (password == null) {
    password = shell.readLine(new Prompt("Password: ", '*'));
}
```

## Custom Types with Converter

```java
@Option(converter = PathConverter.class, description = "Directory path")
private Path directory;

public static class PathConverter implements Converter<Path> {
    @Override
    public Path convert(String input) {
        return Paths.get(input);
    }
}
```

## Short Names

The `shortName` defines the single-character option:

```java
@Option(name = "verbose", shortName = 'v', description = "Verbose mode")
private boolean verbose;
```

Both `--verbose` and `-v` work.
