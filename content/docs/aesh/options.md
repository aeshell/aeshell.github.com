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

This works with both the built-in `AeshOptionParser` and custom `OptionParser` implementations -- the fallback chain runs automatically after any parser returns without setting a value.

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

Negatable options are rendered as a single combined entry using `--[prefix]name` format:

```
Options:
  --[no-]verbose            Enable verbose output
  --[without-]color         Enable color output
```

In the synopsis:
```
Usage: mycommand [--[no-]verbose] [--[without-]color]
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
@CommandDefinition(
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

Use the `:-` separator (bash-style) to provide a fallback value when the variable is not set:

```java
@Option(defaultValue = "${PORT:-8080}", description = "Server port")
private int port;

@Option(defaultValue = "${LOG_LEVEL:-INFO}", description = "Log level")
private String logLevel;

@Option(defaultValue = "${DATABASE_HOST:-localhost}", description = "Database host")
private String dbHost;
```

If `PORT` is not set, the default will be `8080`. If `LOG_LEVEL` is not set, the default will be `INFO`.

### System Properties

Reference Java system properties using the `sys:` prefix or bare name:

```java
@Option(defaultValue = "${sys:user.home}", description = "User home directory")
private String userHome;

@Option(defaultValue = "${user.name}", description = "Current user (sys prop, then env var)")
private String userName;

@Option(defaultValue = "${sys:java.io.tmpdir}", description = "Temp directory")
private String tempDir;

@Option(defaultValue = "${sys:os.name}", description = "Operating system")
private String osName;
```

### System Properties with Fallback

```java
@Option(defaultValue = "${sys:my.app.config:-/etc/myapp/config}", description = "Config path")
private String configPath;
```

### Nested Fallback Chains

Fallback values can themselves be variable expressions, enabling cascading resolution:

```java
// Try env var EDITOR, then sys prop editor.path, then literal "vim"
@Option(defaultValue = "${env:EDITOR:-${sys:editor.path:-vim}}")
private String editor;

// Try env var DB_USER, then system user name
@Option(defaultValue = "${env:DB_USER:-${sys:user.name}}")
private String dbUser;
```

This works for `fallbackValue` too:

```java
// Bare --debug resolves port from env, then config, then literal 4004
@Option(name = "debug", fallbackValue = "${env:DEBUG_PORT:-${sys:debug.port:-4004}}")
private String debug;
```

### Escaping

To use a literal `${...}` string without variable expansion, prefix with an extra `$`:

```java
@Option(defaultValue = "$${NOT_EXPANDED}", description = "Literal ${NOT_EXPANDED}")
private String literal;
```

### Variable Syntax Reference

| Syntax | Meaning | Example |
|--------|---------|---------|
| `${env:VAR}` | Environment variable | `${env:HOME}` |
| `${sys:prop}` | System property | `${sys:user.home}` |
| `${key}` | Sys prop first, then env var | `${HOME}` |
| `${key:-fallback}` | Use fallback if not found | `${PORT:-8080}` |
| `${a:-${b:-c}}` | Nested fallback chain | `${env:A:-${sys:B:-default}}` |
| `$${...}` | Literal (no expansion) | `$${NOT_EXPANDED}` |

Variable resolution applies to `defaultValue` and `fallbackValue` annotation attributes. It runs at option construction time, before `DefaultValueProvider` (which runs at parse time and takes priority over static defaults).

### Combined Example

```java
@CommandDefinition(name = "connect", description = "Connect to database")
public class ConnectCommand implements Command<CommandInvocation> {

    @Option(
        shortName = 'h',
        defaultValue = "${DB_HOST:-localhost}",
        description = "Database host"
    )
    private String host;

    @Option(
        shortName = 'p',
        defaultValue = "${DB_PORT:-5432}",
        description = "Database port"
    )
    private int port;

    @Option(
        shortName = 'u',
        defaultValue = "${env:DB_USER:-${sys:user.name}}",
        description = "Database user (defaults to system user)"
    )
    private String user;

    @Option(
        shortName = 'd',
        defaultValue = "${DB_NAME:-myapp}",
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

For defaults and fallbacks that must be resolved at runtime (e.g., from configuration files, databases, or user profiles), use a `DefaultValueProvider`. The provider participates in the full value resolution chain, handling both "option omitted" and "option present but bare" cases.

### Value Resolution Chain

When determining an option's value, aesh follows this resolution chain (highest to lowest priority):

**When the option is omitted entirely:**

| Priority | Source | When | Example |
|----------|--------|------|---------|
| 1 | **Explicit user value** | `--debug=5005` | Always wins |
| 2 | **Annotation `defaultValue` with resolved env/sys var** | `${env:VAR}` was set | Env var takes priority |
| 3 | **Provider `defaultValue()`** | Option omitted, env var not set | Dynamic default from config |
| 4 | **Annotation `defaultValue`** (static/fallback) | Provider returned null | Static default or `:-` fallback |
| 5 | **Field type default** | Nothing resolved | `null`, `0`, `false` |

**When the option is specified bare (e.g., `--debug` without value):**

| Priority | Source | When | Example |
|----------|--------|------|---------|
| 1 | **Annotation `fallbackValue` with resolved env/sys var** | `${env:VAR}` was set | Env var takes priority |
| 2 | **Provider `fallbackValue()`** | Env var not set | Dynamic fallback from config |
| 3 | **Annotation `fallbackValue`** (static/fallback) | Provider returned null | Static fallback or `:-` portion |
| 4 | **Annotation `defaultValue`** | No fallback available | Legacy fallback |

At each stage, returning `null` (or not setting a value) falls through to the next stage.

{{< callout type="info" >}}
The env var priority only applies when the annotation `defaultValue` or `fallbackValue` uses `${env:...}` or `${sys:...}` syntax **and** the referenced variable is actually set. Options with literal defaults (e.g., `defaultValue = "4004"`) are unaffected — the provider takes priority as before.
{{< /callout >}}

#### Env Var > Config > Hardcoded Default

This resolution chain enables the common "env var > config file > hardcoded default" pattern declaratively:

```java
@CommandDefinition(name = "edit", description = "Edit files",
        defaultValueProvider = AppConfigProvider.class)
public class EditCommand implements Command<CommandInvocation> {

    // Priority: JBANG_EDITOR env var > config "edit.open" > (none)
    @Option(name = "open", defaultValue = "${env:JBANG_EDITOR:-}")
    String editor;
}
```

```bash
# Env var set: uses env var (priority 2)
$ JBANG_EDITOR=code edit file.java    # editor = "code"

# Env var not set, config has edit.open=vim: uses provider (priority 3)
$ edit file.java                       # editor = "vim"

# Env var not set, no config: uses fallback (priority 4)
$ edit file.java                       # editor = ""
```

### Basic DefaultValueProvider

The provider is registered on the command via `@CommandDefinition(defaultValueProvider = ...)`:

```java
public class AppConfigProvider implements DefaultValueProvider {

    @Override
    public String defaultValue(ProcessedOption option) {
        // Called when option is NOT specified at all
        String key = option.parent().name() + "." + option.name();
        return AppConfig.instance().get(key);  // null → falls through to annotation defaultValue
    }
}

@CommandDefinition(name = "run", description = "Run application",
        defaultValueProvider = AppConfigProvider.class)
public class RunCommand implements Command<CommandInvocation> {

    @Option(description = "Debug port", defaultValue = "4004")
    private String debug;
    // Omitted → provider returns config value, or falls through to "4004"
}
```

### Provider Fallback for Bare Flags

The `fallbackValue()` method handles options used without a value (e.g., `--debug` without `=value`). This is useful for three-state options where "bare flag" should resolve from configuration:

```java
public class JBangConfigProvider implements DefaultValueProvider {

    @Override
    public String defaultValue(ProcessedOption option) {
        // Called when option is omitted entirely
        return null;
    }

    @Override
    public String fallbackValue(ProcessedOption option) {
        // Called when option is present but bare (--debug without =value)
        switch (option.name()) {
            case "debug":
                return Configuration.get("run.debug");  // null → falls through to "4004"
            case "jfr":
                return Configuration.get("run.jfr");    // null → falls through to ""
            default:
                return null;  // use annotation fallbackValue
        }
    }
}

@CommandDefinition(name = "run", description = "Run a script",
        defaultValueProvider = JBangConfigProvider.class)
public class RunCommand implements Command<CommandInvocation> {

    @Option(name = "debug", fallbackValue = "4004",
            description = "Enable debugging on port ${DEFAULT-VALUE}")
    String debug;

    @Option(name = "jfr", fallbackValue = "",
            description = "Enable Java Flight Recorder")
    String jfr;
}
```

**How this resolves for `--debug`:**

```bash
# Explicit value — always wins (priority 1)
$ myapp run --debug=5005
# debug = "5005"

# Bare flag, config has run.debug=9999 — provider fallback (priority 2)
$ myapp run --debug
# debug = "9999" (from provider.fallbackValue())

# Bare flag, no config — annotation fallback (priority 3)
$ myapp run --debug
# debug = "4004" (from @Option(fallbackValue = "4004"))

# Option omitted — field default (priority 6)
$ myapp run
# debug = null
```

### Backward Compatibility

The `fallbackValue()` method is a `default` method returning `null`. Existing `DefaultValueProvider` implementations continue to work unchanged — the annotation `fallbackValue` is used as before when the provider doesn't override this method.

### When to Use Each

| Scenario | Use |
|----------|-----|
| Option always has a known default | `@Option(defaultValue = "4004")` |
| Default from env var, higher priority than config | `@Option(defaultValue = "${env:MY_VAR:-}")` |
| Default comes from config at runtime | `DefaultValueProvider.defaultValue()` |
| Bare flag (`--debug`) needs a static value | `@Option(fallbackValue = "4004")` |
| Bare flag needs a value from config | `DefaultValueProvider.fallbackValue()` |
| Env var > config > hardcoded default | `@Option(defaultValue = "${env:VAR:-}")` + provider |

### Custom Parsers and the Fallback Chain

When using a custom `OptionParser` (via `@Option(parser = MyParser.class)`), the value resolution chain applies automatically. If the custom parser decides not to consume a token and returns without calling `addValue()`, the framework applies the fallback chain (provider `fallbackValue()` -> annotation `fallbackValue` -> annotation `defaultValue`).

This separates the **parsing concern** (which tokens to consume) from the **value resolution concern** (what value to use when no tokens were consumed):

```java
public class DebugOptionParser implements OptionParser {
    private static final Pattern PORT = Pattern.compile("\\d+");

    @Override
    public void parse(ParsedLineIterator iter, ProcessedOption option)
            throws OptionParserException {
        // Consume the option name token
        iter.pollParsedWord();

        // Peek at next token -- only consume if it matches a port number
        if (iter.hasNextWord()) {
            String next = iter.peekWord();
            if (!next.startsWith("-") && PORT.matcher(next).matches()) {
                option.addValue(next);
                iter.pollParsedWord();
                return;
            }
        }
        // No addValue() call -- framework applies the fallback chain
    }
}

@CommandDefinition(name = "run", description = "Run script")
public class RunCommand implements Command<CommandInvocation> {

    @Option(name = "debug", parser = DebugOptionParser.class,
            fallbackValue = "4004")
    String debug;

    @Argument
    String script;
}
```

```bash
$ run --debug=5005 test.java   # debug = "5005" (parser consumed)
$ run --debug 5005 test.java   # debug = "5005" (parser consumed)
$ run --debug test.java        # debug = "4004" (parser skipped, fallback applied)
$ run --debug                  # debug = "4004" (parser skipped, fallback applied)
```

The custom parser contract:

| Parser action | Result |
|---|---|
| `option.addValue(x)` | Value is `x` -- no fallback applied |
| `option.addValue("")` | Value is `""` -- no fallback applied (explicit empty) |
| No `addValue()` call | Fallback chain runs (provider -> annotation -> default) |

This eliminates the need for custom parsers to set sentinel values (like `""`) and handle fallback resolution manually.

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

## Optional Values

Options and arguments can be declared as `Optional<T>` to distinguish between "not provided" and "provided with a value":

```java
@CommandDefinition(name = "deploy", description = "Deploy app")
public class DeployCommand implements Command<CommandInvocation> {

    @Option(name = "environment", description = "Target environment")
    Optional<String> environment;

    @Option(name = "replicas", description = "Number of replicas")
    Optional<Integer> replicas;

    @Argument(description = "Application name")
    Optional<String> application;

    @Override
    public CommandResult execute(CommandInvocation ci) {
        if (environment.isPresent()) {
            ci.println("Deploying to " + environment.get());
        } else {
            ci.println("No environment specified, using default");
        }
        ci.println("Replicas: " + replicas.orElse(1));
        return CommandResult.SUCCESS;
    }
}
```

When an option is not provided on the command line, the field is set to `Optional.empty()` (never `null`). When provided, it's wrapped with `Optional.of(value)`.

This works with all option and argument types:

| Declaration | When provided | When not provided |
|-------------|--------------|-------------------|
| `@Option Optional<String> name` | `Optional.of("value")` | `Optional.empty()` |
| `@Option Optional<Integer> count` | `Optional.of(42)` | `Optional.empty()` |
| `@OptionList Optional<List<String>> items` | `Optional.of(["a","b"])` | `Optional.empty()` |
| `@OptionGroup Optional<Map<String,String>> props` | `Optional.of({k=v})` | `Optional.empty()` |
| `@Argument Optional<String> file` | `Optional.of("file.txt")` | `Optional.empty()` |
| `@Arguments Optional<List<String>> files` | `Optional.of(["a","b"])` | `Optional.empty()` |

## Multi-line Descriptions

Option descriptions can contain newlines for richer help output. Continuation lines are automatically indented to align with the description column:

```java
@Option(name = "env", description = "Target environment\nMust be one of: dev, staging, prod")
String environment;
```

Produces:
```
  --env=<env>  Target environment
               Must be one of: dev, staging, prod
```

Text blocks (Java 15+) also work — leading indentation is automatically stripped:

```java
@Option(name = "env", description = """
    Target environment.
    Must be one of: dev, staging, prod.
    Defaults to the value of $APP_ENV.""")
String environment;
```

## Enum Options and Valid Values

When an option uses an enum type, aesh automatically appends the valid values to the help description. You don't need to manually list them -- they stay in sync with the actual enum constants.

```java
public enum Format { TEXT, JSON, XML }

@Option(name = "format", description = "Output format")
Format format;
```

Help output:
```
--format=<format>  Output format. Valid values: text, json, xml
```

If you add a new value to the enum, the help updates automatically.

The auto-append is skipped when the description already contains the values (e.g., via `${COMPLETION-CANDIDATES}`), so there is no duplication:

```java
@Option(name = "format", description = "Output format (${COMPLETION-CANDIDATES})")
Format format;
// Renders: --format=<format>  Output format (text, json, xml)
// "Valid values:" is NOT appended since the values are already shown
```

{{< callout type="info" >}}
Enum values are shown in lowercase in help output. The parser accepts values case-insensitively (e.g., `--format TEXT`, `--format text`, and `--format Text` all work).
{{< /callout >}}

## Description Variables

Option and command descriptions support `${VARIABLE}` placeholders that are resolved at help-render time. This keeps descriptions in sync with annotation values automatically.

### Option-level variables

| Variable | Resolves to | Example |
|----------|-------------|---------|
| `${DEFAULT-VALUE}` | The option's `defaultValue` | `"4004"` |
| `${FALLBACK-VALUE}` | The option's `fallbackValue` | `"4004"` |
| `${COMPLETION-CANDIDATES}` | The option's `allowedValues` (comma-separated) | `"fast, safe"` |

```java
@Option(name = "debug", defaultValue = "4004",
        description = "Enable debugging on port ${DEFAULT-VALUE}")
String debug;
// Renders: --debug=<debug>  Enable debugging on port 4004

@Option(name = "mode", allowedValues = {"fast", "safe"},
        description = "Build mode (${COMPLETION-CANDIDATES})")
String mode;
// Renders: --mode=<mode>    Build mode (fast, safe)

@Option(name = "port", defaultValue = "8080", fallbackValue = "8080",
        optionalValue = true,
        description = "Server port (default: ${DEFAULT-VALUE})")
String port;
// Renders: --port=<port>    Server port (default: 8080)
```

### Command-level variables

These work in both option descriptions and command descriptions, as well as in `HelpSectionProvider` content (header, footer, additional sections):

| Variable | Resolves to |
|----------|-------------|
| `${COMMAND-NAME}` | The current command name (e.g., `run`) |
| `${COMMAND-FULL-NAME}` | The full command path (e.g., `jbang run`) |
| `${ROOT-COMMAND-NAME}` | The root/top-level command name (e.g., `jbang`) |
| `${PARENT-COMMAND-NAME}` | The parent command name (e.g., `jbang`) |
| `${PARENT-COMMAND-FULL-NAME}` | The parent command full path |

```java
@CommandDefinition(name = "run",
        description = "Run a ${ROOT-COMMAND-NAME} script")
public class RunCommand implements Command<CommandInvocation> {
    // Description renders as: "Run a jbang script"
}
```

Unknown variables are left as-is (not replaced).
