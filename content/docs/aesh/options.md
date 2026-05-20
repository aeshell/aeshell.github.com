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
| `fallbackValue` | `String` | `""` | Value when option is specified bare (implies `optionalValue`, not applied when omitted) |
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
| `aliases` | `String[]` | `{}` | Alternative long names for this option |
| `helpGroup` | `String` | `""` | Group heading for this option in help output |
| `allowedValues` | `String[]` | `{}` | Restricts option to a fixed set of valid values |
| `exclusiveWith` | `String[]` | `{}` | Names of mutually exclusive options (without `--` prefix) |
| `visibility` | `OptionVisibility` | `BRIEF` | Controls help and completion visibility (`BRIEF`, `FULL`, `HIDDEN`) |
| `order` | `int` | `Integer.MAX_VALUE` | Explicit help-order position (lower values appear first) |
| `descriptionUrl` | `String` | `""` | URL for option documentation (clickable in supported terminals) |
| `url` | `boolean` | `false` | Treat option value as a URL (rendered as clickable link) |

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

### Boolean vs boolean

Use the primitive `boolean` for simple on/off flags. Use the wrapper `Boolean` when you need three-state semantics:

| Type | Not provided | Provided |
|------|-------------|----------|
| `boolean` | `false` | `true` |
| `Boolean` | `null` | `true` |

This is useful when "not specified" should behave differently from "explicitly disabled":

```java
@Option(hasValue = false, description = "Enable color output")
private Boolean color;  // null = use terminal default, true = force color
```

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

## Fallback Value (Three-State Options)

The `fallbackValue` attribute solves the three-state problem: distinguishing "not specified", "specified without value", and "specified with explicit value". Unlike `optionalValue` + `defaultValue`, the fallback is only applied when the option is specified bare -- not when it's omitted entirely.

### Basic Usage

```java
@CommandDefinition(name = "run", description = "Run application")
public class RunCommand implements Command<CommandInvocation> {

    @Option(name = "debug", fallbackValue = "4004",
            description = "Enable debugging. Default port: ${FALLBACK-VALUE}")
    private String debug;

    @Argument(description = "Script file")
    private String script;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (debug == null) {
            invocation.println("No debugging");
        } else {
            invocation.println("Debugging on port " + debug);
        }
        return CommandResult.SUCCESS;
    }
}
```

| Invocation | `debug` value | Meaning |
|---|---|---|
| `run test.java` | `null` | Not specified -- no debugging |
| `run --debug test.java` | `"4004"` | Bare -- use fallback port |
| `run --debug=5005 test.java` | `"5005"` | Explicit -- use given port |

### How It Works

When `fallbackValue` is set:

1. **Implies `optionalValue = true`** -- You don't need to set both
2. **Bare usage (`--debug`)** applies the fallback value and does NOT consume the next token. The next token (`test.java`) remains available as a positional argument
3. **Explicit value (`--debug=5005`)** uses the given value, same as any normal option
4. **Not specified** leaves the field as `null` (or the `defaultValue` / `DefaultValueProvider` value if configured)

This eliminates the need for custom `OptionParser` implementations to handle the three-state pattern.

### Compared to `optionalValue`

| | `optionalValue` + `defaultValue` | `fallbackValue` |
|---|---|---|
| `--debug` (bare) | Uses `defaultValue` | Uses `fallbackValue` |
| Not specified | Also uses `defaultValue` | `null` (or static `defaultValue`) |
| Can distinguish bare from omitted? | No | Yes |
| Consumes next token? | Yes (ambiguous) | No (only `=` syntax) |

### Programmatic API

```java
ProcessedOptionBuilder.builder()
        .name("debug")
        .type(String.class)
        .fallbackValue("4004")
        .description("Debug port")
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

## Option Aliases

The `aliases` property defines alternative long names for an option. This is useful when migrating from another CLI framework, supporting legacy names, or providing shorter alternatives.

### Basic Usage

```java
@CommandDefinition(name = "runner", description = "Java runner")
public class RunnerCommand implements Command<CommandInvocation> {

    @Option(name = "enableassertions", aliases = {"ea"},
            hasValue = false, description = "Enable assertions")
    private boolean enableAssertions;

    @Option(name = "classpath", aliases = {"cp"},
            description = "Class path")
    private String classpath;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        return CommandResult.SUCCESS;
    }
}
```

All of these are equivalent:

```bash
$ runner --enableassertions --classpath /path
$ runner --ea --cp /path
```

### Help Output

Aliases appear alongside the primary name in help output:

```
Options:
  --enableassertions, --ea      Enable assertions
  --classpath, --cp <value>     Class path
```

### Tab Completion

Both primary names and aliases are offered during tab completion. Typing `--e` and pressing Tab will suggest both `--enableassertions` and `--ea` (if both match the prefix).

### With OptionList

Aliases also work with `@OptionList`:

```java
@OptionList(name = "items", aliases = {"item"},
            description = "Items to process")
private List<String> items;
```

Both `--items a,b,c` and `--item a,b,c` are accepted.

## Help Grouping

The `helpGroup` property organizes options under custom headings in `--help` output. Without it, all options appear under a single "Options:" heading. With `helpGroup`, you can create structured, readable help output by grouping related options together.

### Basic Usage

```java
@CommandDefinition(name = "myapp", description = "My application", generateHelp = true)
public class MyAppCommand implements Command<CommandInvocation> {

    @Option(hasValue = false, description = "Output as JSON", helpGroup = "Output Format")
    private boolean json;

    @Option(hasValue = false, description = "Output as XML", helpGroup = "Output Format")
    private boolean xml;

    @Option(description = "Username", helpGroup = "Authentication")
    private String user;

    @Option(description = "Password", helpGroup = "Authentication")
    private String password;

    @Option(hasValue = false, description = "Verbose output")
    private boolean verbose;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        return CommandResult.SUCCESS;
    }
}
```

Help output:

```
Usage: myapp [<options>]
My application

Output Format:
  --json      Output as JSON
  --xml       Output as XML

Authentication:
  --user      Username
  --password  Password

Options:
  --verbose   Verbose output
```

### How It Works

1. Options with the same `helpGroup` value are displayed together under that heading
2. Named groups appear first, in the order their first option was defined
3. Options without a `helpGroup` (or with an empty value) appear under the default "Options:" heading
4. If all options have a `helpGroup`, the "Options:" heading is omitted entirely
5. Column alignment spans all groups, so descriptions stay aligned across the entire help output

### With OptionList

`helpGroup` also works with `@OptionList`:

```java
@OptionList(description = "Include patterns", helpGroup = "Filters")
private List<String> include;

@OptionList(description = "Exclude patterns", helpGroup = "Filters")
private List<String> exclude;
```

### With Mixins

Mixin options retain their `helpGroup` when included in a command. This makes mixins a natural way to add entire option groups:

```java
public class OutputMixin {
    @Option(hasValue = false, description = "Output as JSON", helpGroup = "Output Format")
    boolean json;

    @Option(hasValue = false, description = "Output as XML", helpGroup = "Output Format")
    boolean xml;
}

@CommandDefinition(name = "report", description = "Generate report", generateHelp = true)
public class ReportCommand implements Command<CommandInvocation> {

    @Mixin
    OutputMixin output;

    @Option(description = "Report title")
    private String title;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        return CommandResult.SUCCESS;
    }
}
```

### Programmatic API

When building commands programmatically, use `ProcessedOptionBuilder`:

```java
ProcessedOptionBuilder.builder()
        .name("json")
        .type(boolean.class)
        .description("Output as JSON")
        .helpGroup("Output Format")
        .build();
```

## Help Option Ordering

Use `order` on options to explicitly control help output order:

```java
@Option(description = "Always first", order = 10)
private boolean ci;

@Option(description = "Shown after --ci", order = 20)
private String profile;
```

Options are ordered by:

1. `order` (lower first)
2. Then by command-level `sortOptions` behavior:
   - `sortOptions = false` (default): declaration order
   - `sortOptions = true`: alphabetical by option name

`order` also works on `@OptionList` and `@OptionGroup`.

## Mutually Exclusive Options

The `exclusiveWith` property declares that two or more options cannot be used together. If a user provides both, a `MutuallyExclusiveOptionException` is thrown during parsing.

### Basic Usage

```java
@CommandDefinition(name = "export", description = "Export data", generateHelp = true)
public class ExportCommand implements Command<CommandInvocation> {

    @Option(hasValue = false, description = "Output as JSON", exclusiveWith = {"xml", "csv"})
    private boolean json;

    @Option(hasValue = false, description = "Output as XML", exclusiveWith = {"json", "csv"})
    private boolean xml;

    @Option(hasValue = false, description = "Output as CSV", exclusiveWith = {"json", "xml"})
    private boolean csv;

    @Option(hasValue = false, description = "Verbose output")
    private boolean verbose;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        return CommandResult.SUCCESS;
    }
}
```

Usage:
```bash
# Valid -- only one format selected
$ export --json
$ export --xml --verbose

# Invalid -- mutually exclusive options used together
$ export --json --xml
Error: Options --json and --xml are mutually exclusive.
```

### How It Works

1. Each option lists the names of options it conflicts with (long names, without the `--` prefix)
2. The relationship should be declared on both sides: if `--json` lists `xml`, then `--xml` should list `json`
3. Validation runs after parsing in both `STRICT` and `VALIDATE` modes, so it is enforced whether you use the interactive console or the runtime API
4. Non-exclusive options (like `--verbose` above) can be freely combined with any exclusive option

### Tab Completion

When an exclusive option has been set, conflicting options are automatically filtered from tab completion. For example, after typing `export --json`, pressing Tab will not suggest `--xml` or `--csv`.

### With OptionList

`exclusiveWith` also works with `@OptionList`:

```java
@OptionList(description = "Items to process", exclusiveWith = {"file"})
private List<String> items;

@Option(description = "Read items from file", exclusiveWith = {"items"})
private String file;
```

### Programmatic API

When building commands programmatically, use `ProcessedOptionBuilder`:

```java
ProcessedOptionBuilder.builder()
        .name("json")
        .type(boolean.class)
        .hasValue(false)
        .exclusiveWith("xml", "csv")
        .build();
```

## Visibility Levels

The `visibility` property controls whether an option appears in `--help` output and tab completion. This is useful for organizing help output by importance — showing essential options by default and revealing advanced or deprecated options only on request.

### Visibility Values

| Level | `--help` | `--help=all` | Tab completion |
|-------|----------|--------------|----------------|
| `BRIEF` (default) | Shown | Shown | Shown |
| `FULL` | Hidden | Shown | Shown |
| `HIDDEN` | Hidden | Hidden | Hidden |

### Basic Usage

```java
@CommandDefinition(name = "serve", description = "Start server", generateHelp = true)
public class ServeCommand implements Command<CommandInvocation> {

    @Option(shortName = 'p', description = "Server port")
    private int port;

    @Option(description = "Bind address")
    private String host;

    @Option(description = "Enable request tracing", visibility = OptionVisibility.FULL)
    private boolean trace;

    @Option(description = "Thread pool size", visibility = OptionVisibility.FULL)
    private int threads;

    @Option(description = "Internal diagnostic token", visibility = OptionVisibility.HIDDEN)
    private String diagnosticToken;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        return CommandResult.SUCCESS;
    }
}
```

Default help shows only essential options:

```bash
$ serve --help
Usage: serve [<options>]
Start server

Options:
  -p, --port      Server port
  --host          Bind address
  -h, --help      Display help (use --help=all for all options)
```

Full help reveals advanced options:

```bash
$ serve --help=all
Usage: serve [<options>]
Start server

Options:
  -p, --port      Server port
  --host          Bind address
  --trace         Enable request tracing
  --threads       Thread pool size
  -h, --help      Display help (use --help=all for all options)
```

The `--diagnosticToken` option never appears in help but still works when typed explicitly.

### When to Use Each Level

- **`BRIEF`** — Options every user needs to know about. This is the default; existing commands are unaffected.
- **`FULL`** — Advanced tuning, debugging, or rarely-used options. Power users can discover them with `--help=all`.
- **`HIDDEN`** — Internal, deprecated, or experimental options that should not be discoverable. They still parse and work when used explicitly.

### Tab Completion

`HIDDEN` options are excluded from tab completion. `BRIEF` and `FULL` options are both offered during completion — visibility only affects help output for `FULL` options.

### With Help Grouping

Visibility works with `helpGroup`. A group heading is only printed if it contains at least one visible option:

```java
@Option(description = "Enable tracing", helpGroup = "Diagnostics",
        visibility = OptionVisibility.FULL)
private boolean trace;

@Option(description = "Profile mode", helpGroup = "Diagnostics",
        visibility = OptionVisibility.FULL)
private boolean profile;
```

With `--help`, the "Diagnostics" group is omitted entirely. With `--help=all`, it appears with both options.

### With OptionList and OptionGroup

Visibility also works with `@OptionList` and `@OptionGroup`:

```java
@OptionList(description = "Debug modules", visibility = OptionVisibility.FULL)
private List<String> debugModules;

@OptionGroup(shortName = 'X', description = "Internal flags",
             visibility = OptionVisibility.HIDDEN)
private Map<String, String> internalFlags;
```

### Programmatic API

When building commands programmatically, use `ProcessedOptionBuilder`:

```java
ProcessedOptionBuilder.builder()
        .name("trace")
        .type(boolean.class)
        .description("Enable request tracing")
        .visibility(OptionVisibility.FULL)
        .build();
```

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

## Allowed Values

The `allowedValues` attribute restricts an option to a fixed set of valid string values. Invalid values are rejected at parse time with a clear error message, and the allowed values are offered as tab completion candidates automatically.

```java
@Option(name = "format",
        allowedValues = { "text", "json", "yaml" },
        description = "Output format")
private String format;
```

```
$ mycmd --format text    # OK
$ mycmd --format xml     # Error: Invalid value 'xml' for option '--format'. Allowed values: text, json, yaml
$ mycmd --format <tab>   # Offers: text, json, yaml
```

When a custom completer is also specified, it takes precedence over the `allowedValues` completion.

`allowedValues` is also available on `@OptionList` to restrict each element in the list:

```java
@OptionList(name = "tags",
            allowedValues = { "v1", "v2", "latest" },
            description = "Deployment tags")
private List<String> tags;
```

For enum-typed options, you don't need to set `allowedValues` manually -- aesh automatically populates `allowedValues` from the enum constants (lowercased). This means enum options get both tab completion and consistent `OptionValidatorException` error messages ("Invalid value 'xyz' for option '--format'. Allowed values: text, json, yaml") without any extra configuration. The check is case-insensitive, so `TEXT`, `text`, and `Text` are all accepted. See also [Converters -- Enum Conversion](../converters#enum-conversion).

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

## File-Typed Options

When an option field is typed as `Resource`, `File`, or `Path`, Aesh automatically provides file path tab completion and type conversion -- no completer or converter needed:

```java
@Option(description = "Output file")
private Resource output;    // automatic file completion + Resource API

@Option(description = "Config file")
private File config;         // automatic file completion + java.io.File

@Option(description = "Data directory")
private Path dataDir;        // automatic file completion + java.nio.file.Path
```

`Resource` is recommended when your command needs to read, write, or inspect files, as it provides a richer API with glob expansion, directory filtering, and I/O streams. See [File and Resource Handling](../resources) for the full API and filtering options.

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
