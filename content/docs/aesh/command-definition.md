---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Command Definition'
weight: 3
---

The `@CommandDefinition` annotation is used to define a command class.

## Required Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | `String` | The command name |

## Optional Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `aliases` | `String[]` | `{}` | Alternative names for the command |
| `description` | `String` | `""` | Command description shown in help |
| `generateHelp` | `boolean` | `false` | Auto-generate `--help` / `-h` boolean flag. Bypasses required argument validation when set. |
| `disableParsing` | `boolean` | `false` | Skip parsing (everything goes to @Arguments) |
| `version` | `String` | `""` | Version string (adds `--version`, `-v` option) |
| `validator` | `Class<? extends CommandValidator>` | `NullCommandValidator.class` | Validator to run before execution |
| `resultHandler` | `Class<? extends ResultHandler>` | `NullResultHandler.class` | Handler to run after execution |
| `activator` | `Class<? extends CommandActivator>` | `NullCommandActivator.class` | Activator to check if command is available |
| `defaultValueProvider` | `Class<? extends DefaultValueProvider>` | `NullDefaultValueProvider.class` | Dynamic default value resolver |
| `stopAtFirstPositional` | `boolean` | `false` | Stop option parsing after the first positional argument |
| `sortOptions` | `boolean` | `false` | Sort help options alphabetically by name (after explicit option order) |
| `helpUrl` | `String` | `""` | URL to documentation (shown in `--help` output) |
| `groupCommands` | `Class<? extends Command>[]` | `{}` | Subcommand classes. Non-empty makes this a [group command](group-commands). Replaces the deprecated `@GroupCommandDefinition`. |
| `helpGroup` | `String` | `""` | Group heading when listed as a subcommand in parent's help |
| `helpSectionProvider` | `Class<? extends HelpSectionProvider>` | `NullHelpSectionProvider.class` | Provider for dynamic help sections |

## Example

```java
@CommandDefinition(
    name = "copy",
    aliases = {"cp"},
    description = "Copy files",
    generateHelp = true
)
public class CopyCommand implements Command<CommandInvocation> {
    
    @Option(shortName = 'r', description = "Recursive copy")
    private boolean recursive;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // implementation
        return CommandResult.SUCCESS;
    }
}
```

## Command Interface

Your command must implement the `Command<T extends CommandInvocation>` interface:

```java
public interface Command<T extends CommandInvocation> {
    CommandResult execute(T commandInvocation) 
        throws CommandException, InterruptedException;
}
```

### CommandResult

Return one of the following:
- `CommandResult.SUCCESS` - Command completed successfully
- `CommandResult.FAILURE` - Command failed
- `CommandResult.RETURN` - Return from current subcommand

## Dynamic Default Value Provider

The `defaultValueProvider` attribute specifies a class that resolves option defaults at runtime. This is useful when defaults come from configuration files, environment variables, or other external sources not known at compile time.

### Implementing a Provider

Create a class that implements `DefaultValueProvider`:

```java
public class ConfigDefaultProvider implements DefaultValueProvider {

    @Override
    public String defaultValue(ProcessedOption option) {
        // Use option.name() and option.parent().name() to build a config key
        String key = option.parent().name() + "." + option.name();
        return Configuration.get(key);  // returns null if not configured
    }
}
```

### Registering the Provider

```java
@CommandDefinition(
    name = "init",
    description = "Initialize a project",
    defaultValueProvider = ConfigDefaultProvider.class
)
public class InitCommand implements Command<CommandInvocation> {

    @Option(defaultValue = "hello")
    private String template;

    @Option
    private String editor;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Template: " + template);
        invocation.println("Editor: " + editor);
        return CommandResult.SUCCESS;
    }
}
```

### Value Precedence

When determining option values, the precedence is (highest to lowest):

1. **User-provided value** -- Explicitly set on the command line
2. **Dynamic default** -- Returned by the `DefaultValueProvider` (if non-null)
3. **Static default** -- The `defaultValue` from the annotation
4. **null** -- If nothing else is set

If the provider returns `null` for an option, aesh falls back to the static `defaultValue`. This lets you use annotation defaults as fallbacks:

```java
// Provider returns "from-config" for template -> uses "from-config"
// Provider returns null for editor -> falls back to static default "vi"
@Option(defaultValue = "vi")
private String editor;
```

See [Options - Dynamic Default Values](/docs/aesh/options#dynamic-default-values) for more details.

### Registry-Level Provider

Instead of repeating `defaultValueProvider = MyProvider.class` on every command, you can set a single provider at the registry or runtime level:

```java
// Via AeshRuntimeRunner
AeshRuntimeRunner.builder()
    .command(MyApp.class)
    .defaultValueProvider(new ConfigDefaultProvider())
    .execute();

// Via AeshCommandRegistryBuilder
CommandRegistry registry = AeshCommandRegistryBuilder.builder()
    .defaultValueProvider(new ConfigDefaultProvider())
    .command(RunCommand.class)
    .command(BuildCommand.class)
    .create();

// Via AeshCommandRuntimeBuilder
CommandRuntime runtime = AeshCommandRuntimeBuilder.builder()
    .commandRegistry(registry)
    .defaultValueProvider(new ConfigDefaultProvider())
    .build();
```

The registry-level provider applies to all commands (including group command children) that don't declare their own per-command provider. Per-command `@CommandDefinition(defaultValueProvider = ...)` takes precedence when set.

## Stop at First Positional

When `stopAtFirstPositional = true`, option parsing stops as soon as the first positional argument is consumed. All remaining tokens are treated as positional arguments, even if they look like options.

This is useful for commands that pass arguments through to another process:

```java
@CommandDefinition(
    name = "run",
    description = "Run a script",
    stopAtFirstPositional = true,
    generateHelp = true
)
public class RunCommand implements Command<CommandInvocation> {

    @Option(hasValue = false, description = "Enable verbose output")
    private boolean verbose;

    @Arguments
    private List<String> args;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // args contains the script name and all arguments after it
        return CommandResult.SUCCESS;
    }
}
```

With the command above:

| Input | Result |
|-------|--------|
| `run --verbose myscript.java` | `verbose=true`, args=`[myscript.java]` |
| `run --verbose myscript.java -Dfoo=bar --help` | `verbose=true`, args=`[myscript.java, -Dfoo=bar, --help]` |
| `run myscript.java --verbose` | `verbose=false`, args=`[myscript.java, --verbose]` |
| `run --help` | Displays help output |

Note that `--help` (and `--version`) before the first positional still work normally when `generateHelp = true`. Only tokens after the first positional are treated as passthrough arguments.

## Option Ordering in Help

Use `sortOptions` to control how options are ordered in help output:

- `sortOptions = false` (default): options are ordered by explicit option `order`, then declaration order
- `sortOptions = true`: options are ordered by explicit option `order`, then alphabetical name

```java
@CommandDefinition(
    name = "build",
    description = "Build project",
    sortOptions = true,
    generateHelp = true
)
public class BuildCommand implements Command<CommandInvocation> {

    @Option(description = "Always first", order = 10)
    private boolean ci;

    @Option(description = "Build profile", order = 20)
    private String profile;

    @Option(description = "Clean output")
    private boolean clean;

    @Option(description = "Verbose output")
    private boolean verbose;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        return CommandResult.SUCCESS;
    }
}
```

The `order` attribute is defined on `@Option`, `@OptionList`, and `@OptionGroup`.

## Help Output Layout

When `generateHelp = true`, aesh produces styled help output with the following structure:

```
Description text
Usage: command [-hv] [--config=<config>] [--verbose | --quiet] [COMMAND]

  Examples or header text (from HelpSectionProvider)

Options:
  -h, --help              Display this help and exit
  --config=<config>       Path to config file
  --[no-]verbose          Enable verbose output
      <scriptOrFile>      Script file to run
      [<userParams>...]   Additional parameters

command commands:
    run       Run a script
    build     Build a project

Footer text (from HelpSectionProvider)
```

Key formatting features:
- **ANSI colors**: option names in yellow, value placeholders in cyan, command name in bold
- **Detailed synopsis**: grouped boolean flags `[-hv]`, value options `[--config=<config>]`, mutually exclusive pipes `[--verbose | --quiet]`, `[COMMAND]` for group commands
- **Synopsis wrapping**: wraps at 80 columns with continuation indentation
- **Option column cap**: option names wider than 24 characters wrap the description to the next line
- **Negatable options**: rendered as `--[no-]name` (single entry, not two)
- **Value placeholders**: options that accept values show `=<name>` (using option name or `argument` attribute)
- **`@OptionGroup` key=value**: rendered as `=<key=value>` to show the expected syntax
- **Inline positional arguments**: shown after options, not in separate sections
- **`--help` bypasses validation**: required arguments are not validated when `--help` is used
- All ANSI styling is disabled when `ansiMode = false`

### CommandInvocation

Provides access to:
- `println(String)` - Output text to the console
- `print(String)` - Output text without newline
- `stop()` - Stop the console
- `getShell()` - Access the shell
- `getHelpInfo(String)` - Get help text

## Help Group for Subcommands

The `helpGroup` property on `@CommandDefinition` controls how a subcommand appears in its parent's help output. Subcommands with the same `helpGroup` value are displayed together under that heading.

### Basic Usage

```java
@CommandDefinition(
    name = "cli",
    description = "My CLI tool",
    generateHelp = true,
    groupCommands = {
        BuildCommand.class, TestCommand.class,
        InstallCommand.class, PublishCommand.class,
        InfoCommand.class, VersionCommand.class
    }
)
public class CliCommand implements GroupCommand<CommandInvocation> {
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        return CommandResult.SUCCESS;
    }
}

@CommandDefinition(name = "build", description = "Build the project", helpGroup = "Build")
public class BuildCommand implements Command<CommandInvocation> { /* ... */ }

@CommandDefinition(name = "test", description = "Run tests", helpGroup = "Build")
public class TestCommand implements Command<CommandInvocation> { /* ... */ }

@CommandDefinition(name = "install", description = "Install dependencies", helpGroup = "Publish")
public class InstallCommand implements Command<CommandInvocation> { /* ... */ }

@CommandDefinition(name = "publish", description = "Publish package", helpGroup = "Publish")
public class PublishCommand implements Command<CommandInvocation> { /* ... */ }

@CommandDefinition(name = "info", description = "Show project info")
public class InfoCommand implements Command<CommandInvocation> { /* ... */ }

@CommandDefinition(name = "version", description = "Show version")
public class VersionCommand implements Command<CommandInvocation> { /* ... */ }
```

Help output:
```
Usage: cli [<options>]
My CLI tool

Build:
    build     Build the project
    test      Run tests

Publish:
    install   Install dependencies
    publish   Publish package

Other:
    info      Show project info
    version   Show version
```

### How It Works

1. Subcommands with the same `helpGroup` value are grouped under that heading
2. Named groups appear first, in the order their first subcommand was defined
3. Subcommands without a `helpGroup` appear under a default heading
4. If all subcommands have a `helpGroup`, there is no default group

This is separate from the `helpGroup` property on `@Option`, which groups options within a single command's help output. See [Options - Help Grouping](/docs/aesh/options#help-grouping).

## Help Section Provider

The `helpSectionProvider` property lets you dynamically add sections to a command's help output at render time. This is useful for showing external plugins, aliases, or dynamically discovered commands without statically defining them.

### Implementing a Provider

Create a class that implements `HelpSectionProvider`:

```java
public class PluginHelpProvider implements HelpSectionProvider {

    @Override
    public Map<String, List<HelpEntry>> getAdditionalSections() {
        Map<String, List<HelpEntry>> sections = new LinkedHashMap<>();

        // Discover plugins at runtime
        List<HelpEntry> plugins = new ArrayList<>();
        plugins.add(new HelpEntry("docker", "Docker integration plugin"));
        plugins.add(new HelpEntry("k8s", "Kubernetes deployment plugin"));
        sections.put("Plugins", plugins);

        return sections;
    }
}
```

### Registering the Provider

```java
@CommandDefinition(
    name = "app",
    description = "My application",
    generateHelp = true,
    groupCommands = {RunCommand.class, BuildCommand.class},
    helpSectionProvider = PluginHelpProvider.class
)
public class AppCommand implements GroupCommand<CommandInvocation> {
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        return CommandResult.SUCCESS;
    }
}
```

Help output:
```
Usage: app [<options>]
My application

app commands:
    run       Run the application
    build     Build the project

Plugins:
    docker    Docker integration plugin
    k8s       Kubernetes deployment plugin
```

### Merging with helpGroup

If a provider section name matches an existing `helpGroup` from statically defined subcommands, the entries are appended to that group rather than creating a duplicate heading.

### Header and Footer

`HelpSectionProvider` can also supply header text (shown before the synopsis) and footer text (shown after everything). Both are `default` methods returning `String`, so existing implementations are unaffected:

```java
public class JBangHelpProvider implements HelpSectionProvider {

    @Override
    public String getHeader() {
        return "jbang - Unleash the power of Java\n"
             + "  jbang init hello.java        (initialize a script)\n"
             + "  jbang hello.java [args...]    (run a .java file)";
    }

    @Override
    public String getFooter() {
        return "See https://jbang.dev for more info";
    }

    @Override
    public Map<String, List<HelpEntry>> getAdditionalSections() {
        return Collections.emptyMap();
    }
}
```

Help output:
```
jbang - Unleash the power of Java
  jbang init hello.java        (initialize a script)
  jbang hello.java [args...]   (run a .java file)
Usage: jbang [<options>]
JBang tool

Options:
  ...

See https://jbang.dev for more info
```

Both methods return `null` by default (no header/footer). The text can contain newline characters for multi-line content.

### Key Points

1. **Zero startup cost** -- The provider class is stored as a reference and only instantiated when help is rendered
2. **Works with `@CommandDefinition`** (including group commands with `groupCommands`)
3. **HelpEntry** is a simple value class with `name()` and `description()` (description is optional)
4. **Return an empty map** (not null) if there are no additional sections to show
5. **Header** appears before the synopsis, **footer** appears after all other content

## NO_COLOR Support

Aesh respects the [NO_COLOR](https://no-color.org) standard. When the `NO_COLOR` environment variable is set (to any value), all ANSI color and styling codes are suppressed in help output:

```bash
NO_COLOR=1 myapp --help   # plain text, no ANSI codes
```

This affects synopsis (bold command, yellow options, cyan placeholders), option formatting, and any other framework-generated styled text. The setting can also be controlled programmatically via `parser.updateAnsiMode(false)`.
