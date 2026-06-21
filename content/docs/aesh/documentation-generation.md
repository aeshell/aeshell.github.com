---
date: '2026-05-29T12:00:00+02:00'
draft: false
title: 'Documentation Generation'
weight: 18
---

Aesh can generate documentation files from your command metadata in AsciiDoc, Markdown, or AI Skill format. This produces one page per command/subcommand, suitable for project documentation sites, man page alternatives, API reference, or AI agent skill definitions.

## Quick Start

The simplest way to generate documentation is with the built-in `--aesh-doc` flag:

```bash
# Generate AsciiDoc to stdout (default)
$ myapp --aesh-doc

# Generate Markdown
$ myapp --aesh-doc markdown

# Generate AI skill documentation (agentskills.io format)
$ myapp --aesh-doc skill

# Save to a file
$ myapp --aesh-doc asciidoc > docs/myapp.adoc
```

No code changes needed — this works automatically with the standard `AeshRuntimeRunner` pattern.

### Per-Command Help Format

When `generateHelp = true` is set on a command, users can request formatted documentation directly from `--help` by appending a format value:

```bash
# Terminal help (unchanged default)
$ myapp --help

# AI skill-format documentation
$ myapp --help=skill

# Markdown documentation
$ myapp --help=markdown
$ myapp --help=md

# AsciiDoc documentation
$ myapp --help=asciidoc
$ myapp --help=adoc

# Works for subcommands too
$ myapp run --help=skill
```

This generates documentation for the specific command (or subcommand) using the same `DocumentationGenerator` renderers as `--aesh-doc`, but scoped to a single command rather than the entire CLI. The `--help` (bare) and `--help=all` behaviors are unchanged.

## Programmatic API

For build-time generation or more control, use the `DocumentationGenerator` builder:

```java
import org.aesh.command.DocFormat;
import org.aesh.util.doc.DocumentationGenerator;

// Generate one file per command/subcommand
DocumentationGenerator.builder()
    .commandClass(MyCommand.class)
    .outputDir(new File("docs/pages"))
    .format(DocFormat.ASCIIDOC)
    .generate();
```

### Antora Integration

For [Antora](https://antora.org/) documentation sites, set a cross-reference prefix and generate a navigation file:

```java
DocumentationGenerator.builder()
    .commandClass(MyCommand.class)
    .outputDir(new File("docs/modules/cli/pages"))
    .crossRefPrefix("myapp:cli:")
    .navFile(new File("docs/modules/cli/nav.adoc"))
    .format(DocFormat.ASCIIDOC)
    .generate();
```

This produces:
- One `.adoc` file per command/subcommand (e.g., `myapp.adoc`, `myapp-run.adoc`)
- A `nav.adoc` with Antora-compatible `xref:` links
- Cross-reference links between pages using the configured prefix

### Single Command Output

Generate documentation for a single command as a string:

```java
String doc = DocumentationGenerator.builder()
    .commandClass(MyCommand.class)
    .format(DocFormat.MARKDOWN)
    .generateSingle();
```

## Generated Sections

Each generated page includes:

| Section | Content |
|---------|---------|
| **NAME** | Command name and description |
| **SYNOPSIS** | Formatted usage with short flag clusters, negatable options, exclusive pipes, value placeholders, and `[COMMAND]` placeholder for groups. Consistent across all formats and `--help`. |
| **DESCRIPTION** | Command description from the annotation |
| **OPTIONS** | All visible options with short/long names, value placeholders, defaults, aliases, and negatable display |
| **ARGUMENTS** | Positional parameters with labels |
| **COMMANDS** | Subcommands for group commands, with cross-reference links |

### HelpSectionProvider Content

If your command uses a [`HelpSectionProvider`](../advanced-topics), its content is included:

- **Header** — inserted between DESCRIPTION and OPTIONS
- **Additional sections** — rendered as separate sections (e.g., EXAMPLES)
- **Footer** — appended at the end

```java
@CommandDefinition(name = "deploy", description = "Deploy app",
    helpSectionProvider = DeployHelpProvider.class)
public class DeployCommand implements Command<CommandInvocation> { ... }

public class DeployHelpProvider implements HelpSectionProvider {
    @Override
    public String getHeader() {
        return "Deploys the application to the configured environment.";
    }

    @Override
    public Map<String, List<HelpEntry>> getAdditionalSections() {
        return Map.of("Examples", List.of(
            new HelpEntry("deploy --env prod myapp", "Deploy to production"),
            new HelpEntry("deploy --force myapp", "Force redeploy")));
    }

    @Override
    public String getFooter() {
        return "Report bugs to https://github.com/example/deploy/issues";
    }
}
```

### Option Rendering Features

The generator handles all option features:

- **Aliases** — `--ea` shown alongside `--enableassertions`
- **Negatable options** — displayed as `--[no-]verbose`
- **Value placeholders** — `--config=<config>`, `<key>=<value>` for option groups
- **Default values** — shown as `Default: \`dev\``
- **Hidden option filtering** — `OptionVisibility.HIDDEN` options are excluded
- **Multi-line descriptions** — text blocks and `\n` in descriptions render properly

## Output Formats

### AsciiDoc

```asciidoc
= DEPLOY

== NAME

deploy -- Deploy an application

== SYNOPSIS

[source]
----
deploy [-f] [-e=<environment>] [-D<key>=<value>] <application>
----

== OPTIONS

*-e*, *--environment*=<environment>::
Target environment
+
Default: `dev`

*-f*, *--force*::
Force deployment
```

### Markdown

```markdown
# DEPLOY

## NAME

deploy -- Deploy an application

## SYNOPSIS

\`\`\`
deploy [-f] [-e=<environment>] [-D<key>=<value>] <application>
\`\`\`

## OPTIONS

### `-e`, `--environment=<environment>`

Target environment

Default: `dev`

### `-f`, `--force`

Force deployment
```

### AI Skill Format

The `skill` format produces structured Markdown with YAML front matter following the [agentskills.io specification](https://agentskills.io/specification), optimized for AI agent consumption:

```markdown
---
name: deploy
description: >-
  Deploy applications to cloud environments. Use when the user wants
  to ship code to production or staging.
license: Apache-2.0
---

## Usage

```
deploy [-f] [-e=<environment>] [-D <key>=<value>] <application>
```

## Options

| Option | Type | Required | Default | Description |
|--------|------|----------|---------|-------------|
| `--environment`, `-e` | String | no | `dev` | Target environment |
| `--force`, `-f` | boolean | no | - | Force deployment |
| `--secret` | boolean | no | - | Internal flag |

## Arguments

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `<application>` | String | yes | Application name |
```

Key differences from the regular Markdown format:

- **YAML front matter** with `name` and `description` fields per agentskills.io spec
- **Options in a table** with Type, Required, Default columns (structured, easy for LLMs to parse)
- **All options included** including `HIDDEN` options (AI agents may need them)
- **Subcommands in a table** (not links to separate files)
- **Single document** with all subcommand details inline
- **Format-specific content** from `HelpSectionProvider` (see below)

## Format-Aware HelpSectionProvider

`HelpSectionProvider` supports format-specific overloads, allowing commands to provide different content for different output formats. This is especially useful for AI skill documentation where descriptions and examples should be tailored for agent consumption.

```java
public class DeployHelpProvider implements HelpSectionProvider {

    // Default sections (used by --help, asciidoc, markdown)
    @Override
    public Map<String, List<HelpEntry>> getAdditionalSections() {
        return Map.of("Plugins", List.of(
            new HelpEntry("maven", "Maven build integration")));
    }

    // Format-specific sections (used by --aesh-doc skill)
    @Override
    public Map<String, List<HelpEntry>> getAdditionalSections(DocFormat format) {
        if (format == DocFormat.SKILL) {
            return Map.of("Examples", List.of(
                new HelpEntry("deploy --env prod myapp", "Deploy to production"),
                new HelpEntry("deploy --force myapp", "Force redeploy")));
        }
        return getAdditionalSections();
    }

    // Override the annotation description for skill format
    @Override
    public String getDescription(DocFormat format) {
        if (format == DocFormat.SKILL) {
            return "Deploy applications to cloud environments. "
                 + "Use when the user wants to ship code to production or staging.";
        }
        return null; // use @CommandDefinition(description=...) for other formats
    }

    // Additional YAML front matter for skill format
    @Override
    public Map<String, String> getFrontMatter(DocFormat format) {
        if (format == DocFormat.SKILL) {
            return Map.of(
                "license", "Apache-2.0",
                "compatibility", "Requires Java 17+");
        }
        return null;
    }
}
```

### Available Format-Aware Methods

All methods have defaults that delegate to the existing parameterless versions, so existing implementations are fully backward compatible.

| Method | Purpose |
|--------|---------|
| `getAdditionalSections(DocFormat)` | Format-specific help sections |
| `getHeader(DocFormat)` | Format-specific header text |
| `getFooter(DocFormat)` | Format-specific footer text |
| `getDescription(DocFormat)` | Override the annotation description per format |
| `getFrontMatter(DocFormat)` | Additional YAML front matter key-value pairs (for SKILL format) |

The `DocFormat` enum values are: `ASCIIDOC`, `MARKDOWN`, `SKILL`.

{{< callout type="info" >}}
The runtime `--help` path does not use `DocFormat` -- it always calls the parameterless methods. Format-aware methods are only used during documentation generation (`--aesh-doc` or the programmatic API).
{{< /callout >}}

## Multi-line Descriptions

Descriptions containing newlines are rendered properly in both `--help` output and generated documentation. This includes text blocks (Java 15+):

```java
@Option(name = "env", description = """
    Target environment.
    Must be one of: dev, staging, prod.
    Defaults to the value of $APP_ENV.""")
```

Leading indentation from text blocks is automatically stripped.
