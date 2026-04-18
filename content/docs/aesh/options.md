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
| `optionalValue` | `boolean` | `false` | Allow option with or without a value (arity 0..1) |
| `defaultValue` | `String[]` | `{}` | Default values |
| `askIfNotSet` | `boolean` | `false` | Prompt user if not set |
| `overrideRequired` | `boolean` | `false` | Override required validation |
| `acceptNameWithoutDashes` | `boolean` | `false` | Allow option name without `--` prefix |
| `negatable` | `boolean` | `false` | Enable `--no-{name}` form for boolean options |
| `negationPrefix` | `String` | `"no-"` | Prefix for negated form (e.g., `"without-"`) |
| `inherited` | `boolean` | `false` | Make option available to subcommands |
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

## Optional Value Options

Optional value options (arity 0..1) can be used both as a flag (no value) and as a valued option. When used without a value, the `defaultValue` is applied. When used with a value, that value is used.

This is useful for options like `--debug` (enable with default port) vs `--debug 5005` (enable with specific port).

### Basic Usage

```java
@CommandDefinition(name = "run", description = "Run application")
public class RunCommand implements Command<CommandInvocation> {

    @Option(shortName = 'd', optionalValue = true, defaultValue = "4004",
            description = "Enable debugging, optionally with a specific port")
    private String debug;

    @Option(optionalValue = true, defaultValue = "default",
            description = "Enable JFR recording")
    private String jfr;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (debug != null) {
            invocation.println("Debugging on port " + debug);
        }
        return CommandResult.SUCCESS;
    }
}
```

Usage:
```bash
# Flag-only: uses defaultValue "4004"
$ run --debug
Debugging on port 4004

# With value via space
$ run --debug 5005
Debugging on port 5005

# With value via equals
$ run --debug=6006
Debugging on port 6006

# Short name
$ run -d
Debugging on port 4004

# Short name with value (no space)
$ run -d8080
Debugging on port 8080

# Combined with other options -- debug uses default, jfr gets value
$ run --debug --jfr=filename=recording.jfr
Debugging on port 4004
```

### How It Works

When the parser encounters an optional-value option:

1. If the next token is another option (e.g., `--debug --verbose`), no value is consumed and the `defaultValue` is used
2. If the next token is a regular value (e.g., `--debug 5005`), it is consumed as the option's value
3. If there are no more tokens (e.g., `--debug` at end of input), the `defaultValue` is used
4. If the value is provided via `=` (e.g., `--debug=5005`), it is always consumed

### Important Notes

1. **Requires `hasValue = true`** (the default) -- `optionalValue` is only valid for options that accept values, not boolean flags.

2. **Works with any type** -- The option field can be `String`, `Integer`, or any type with a registered converter.

3. **`defaultValue` is always applied** -- Whether the option is provided or not, if `defaultValue` is set, it will be used as a fallback. If you need to distinguish "not provided" from "provided without value", omit `defaultValue` and check for `null`.

### Programmatic API

When building commands programmatically (e.g., with the annotation processor), use `ProcessedOptionBuilder`:

```java
ProcessedOptionBuilder.builder()
        .name("debug")
        .shortName('d')
        .type(String.class)
        .optionalValue(true)
        .addDefaultValue("4004")
        .description("Enable debugging")
        .build();
```

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

## Inherited Options

Inherited options are automatically available to all subcommands of a group command. Mark an option with `inherited = true` on the parent, and subcommands can use that option even without declaring it themselves.

### Basic Usage

```java
@GroupCommandDefinition(
    name = "project",
    description = "Project management",
    groupCommands = {BuildCommand.class, TestCommand.class}
)
public class ProjectCommand implements Command<CommandInvocation> {

    @Option(name = "verbose", hasValue = false, inherited = true)
    private boolean verbose;

    @Option(name = "config", inherited = true)
    private String configFile;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        return CommandResult.SUCCESS;
    }
}

@CommandDefinition(name = "build", description = "Build the project")
public class BuildCommand implements Command<CommandInvocation> {

    // Plain fields with matching names -- auto-populated from parent's inherited options
    private boolean verbose;
    private String configFile;

    @Option(description = "Build target")
    private String target;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (verbose) {
            invocation.println("[VERBOSE] Config: " + configFile);
        }
        invocation.println("Building " + target);
        return CommandResult.SUCCESS;
    }
}
```

The inherited option can be placed either before or after the subcommand name:

```bash
# Option before the subcommand -- parsed by the parent
$ project --verbose --config app.yml build --target release

# Option after the subcommand -- recognized from the parent
$ project build --verbose --config app.yml --target release

# Both forms produce the same result
```

### How It Works

1. When the parser encounters an unknown option on a child command, it checks the parent's options for an inherited match
2. If found, the option is parsed using the parent's definition (the value is stored on the parent)
3. After both parent and child are populated, inherited values are propagated into matching fields on the child command
4. If the child has no matching field, the value remains on the parent (accessible via `@ParentCommand`)

### Field Matching

For auto-propagation, the child command needs a field with the **same name** as the parent's inherited option field. The field does not need to be annotated with `@Option`:

```java
// Parent
@Option(name = "verbose", hasValue = false, inherited = true)
private boolean verbose;

// Child -- plain field, same name
private boolean verbose;  // receives the inherited value
```

If the child also declares the option with `@Option`, it works as a regular option -- `searchAllOptions` finds it directly on the child, so no inheritance lookup is needed.

### Best Practices

1. **Use for common flags** -- Options like `--verbose`, `--debug`, `--config` that apply to all subcommands are good candidates.

2. **Don't overuse** -- Only mark options as inherited if subcommands actually need them.

3. **Match field names** -- For auto-propagation, the child's field name must match the parent's field name.

4. **Document inheritance** -- Let users know which options are inherited in your help text.

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

### Dynamic Default Values

For defaults that must be resolved at runtime (e.g., from configuration files, databases, or user profiles), use a `DefaultValueProvider`. The provider is registered on the command, not on individual options:

```java
public class AppConfigProvider implements DefaultValueProvider {

    @Override
    public String defaultValue(ProcessedOption option) {
        // Build a hierarchical key from command + option name
        String key = option.parent().name() + "." + option.name();
        return AppConfig.instance().get(key);  // null if not configured
    }
}

@CommandDefinition(
    name = "run",
    description = "Run application",
    defaultValueProvider = AppConfigProvider.class
)
public class RunCommand implements Command<CommandInvocation> {

    @Option(description = "Debug port", defaultValue = "4004")
    private String debug;

    @Option(description = "JFR settings")
    private String jfr;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // debug = config value if set, else "4004", unless user provided a value
        return CommandResult.SUCCESS;
    }
}
```

If the provider returns `null` for an option, the static `defaultValue` from the annotation is used as a fallback. See [Command Definition - Dynamic Default Value Provider](/docs/aesh/command-definition#dynamic-default-value-provider) for more details.

### Priority Order

When determining option values, Æsh uses this priority (highest to lowest):

1. **Command-line argument** - Explicitly provided by user
2. **Dynamic default** - From `DefaultValueProvider` (if non-null)
3. **Static default / Environment variable / System property** - From `defaultValue` annotation
4. **Fallback value** - Value after `:` in the `defaultValue` expression
5. **Field initializer** - Java field initialization value
6. **null** - If nothing else is set

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

## Ask If Not Set

The `askIfNotSet` property causes Æsh to interactively prompt the user for a value when the option was not provided on the command line. This works with `@Option`, `@Argument`, and `@Arguments`.

### Basic Usage

```java
@CommandDefinition(name = "connect", description = "Connect to server")
public class ConnectCommand implements Command<CommandInvocation> {

    @Option(required = true, description = "Server hostname")
    private String host;

    @Option(askIfNotSet = true, description = "Username")
    private String username;

    @Option(askIfNotSet = true, description = "Password")
    private String password;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Connecting as " + username + "@" + host);
        return CommandResult.SUCCESS;
    }
}
```

If the user runs `connect --host myserver` without `--username` or `--password`, Æsh prompts for each missing value before executing:

```
$ connect --host myserver
Option username, is not set, please provide a value: alice
Option password, is not set, please provide a value: secret123
Connecting as alice@myserver
```

For `@Argument` and `@Arguments`, the prompt reads: `Argument(s) is not set, please provide a value:`.

### Interaction with defaultValue

If a `defaultValue` is set, `askIfNotSet` is ignored -- the default is used instead of prompting. This is by design: if you have a sensible default, there is no need to interrupt the user.

```java
// Will NOT prompt -- the default "8080" is used instead
@Option(askIfNotSet = true, defaultValue = "8080", description = "Port")
private int port;

// WILL prompt -- no default value
@Option(askIfNotSet = true, description = "API key")
private String apiKey;
```

### Interaction with Selectors

When an option has both `askIfNotSet = true` and a `selector` type, the selector UI is used instead of a plain text prompt. See [Selectors](/docs/aesh/selectors) for details.

### Interaction with Help

When `generateHelp = true` and the user runs `command --help`, `askIfNotSet` prompts are skipped. Help output is shown without interruption.

## Override Required

The `overrideRequired` property allows a single option to bypass validation of **all** other required options when it is used. The typical use case is `--help` or `--version` flags that should work without requiring the user to provide all mandatory options.

### Basic Usage

```java
@CommandDefinition(name = "deploy", description = "Deploy application",
                   validator = DeployValidator.class)
public class DeployCommand implements Command<CommandInvocation> {

    @Option(required = true, description = "Target environment")
    private String environment;

    @Option(required = true, description = "Application version")
    private String version;

    @Option(shortName = 'h', hasValue = false, overrideRequired = true,
            description = "Show help")
    private boolean help;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (help) {
            invocation.println(invocation.getHelpInfo("deploy"));
            return CommandResult.SUCCESS;
        }
        invocation.println("Deploying " + version + " to " + environment);
        return CommandResult.SUCCESS;
    }
}
```

Without `overrideRequired`, running `deploy -h` would fail because `--environment` and `--version` are required. With `overrideRequired = true` on the `--help` flag, the required checks are skipped when `-h` is used:

```bash
# Works -- overrideRequired bypasses required checks
$ deploy -h
Usage: deploy [options]
  ...

# Still validates normally when -h is NOT used
$ deploy
Error: Option: --environment is required for this command
```

### What Gets Bypassed

When any option with `overrideRequired = true` is set by the user:

1. **Required option checks** are skipped for all options
2. **Command validator** (`CommandValidator`) is not called
3. **Selector prompts** are skipped

This is an all-or-nothing mechanism: if any `overrideRequired` option is active, all required checks are bypassed.

## Accept Name Without Dashes

The `acceptNameWithoutDashes` property allows users to specify the long option name without the `--` prefix. This only applies to long option names, not short names.

### Basic Usage

```java
@CommandDefinition(name = "test", description = "Run tests")
public class TestCommand implements Command<CommandInvocation> {

    @Option(name = "verbose", acceptNameWithoutDashes = true,
            hasValue = false, description = "Verbose output")
    private boolean verbose;

    @Option(name = "output", acceptNameWithoutDashes = true,
            description = "Output file")
    private String output;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("verbose=" + verbose + ", output=" + output);
        return CommandResult.SUCCESS;
    }
}
```

All of these are equivalent:

```bash
# Standard form
$ test --verbose --output results.txt

# Without dashes
$ test verbose output results.txt

# Mixed
$ test verbose --output results.txt
```

### Tab Completion

When `acceptNameWithoutDashes = true`, tab completion shows the option name without `--`:

```
$ test <TAB>
verbose    output
```

When `acceptNameWithoutDashes = false` (the default), completion shows the standard form:

```
$ test <TAB>
--verbose    --output
```

### Use Cases

This feature is useful for building CLIs where the sub-command style is preferred over traditional option syntax:

```java
@CommandDefinition(name = "config", description = "Configuration manager")
public class ConfigCommand implements Command<CommandInvocation> {

    @Option(name = "key", acceptNameWithoutDashes = true,
            required = true, description = "Configuration key")
    private String key;

    @Option(name = "value", acceptNameWithoutDashes = true,
            description = "Value to set")
    private String value;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (value != null) {
            invocation.println("Set " + key + " = " + value);
        } else {
            invocation.println("Get " + key);
        }
        return CommandResult.SUCCESS;
    }
}
```

```bash
$ config key database.host value localhost
Set database.host = localhost
```
