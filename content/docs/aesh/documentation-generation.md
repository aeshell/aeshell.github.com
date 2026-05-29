---
date: '2026-05-29T12:00:00+02:00'
draft: false
title: 'Documentation Generation'
weight: 18
---

Aesh can generate documentation files from your command metadata in AsciiDoc or Markdown format. This produces one page per command/subcommand, suitable for project documentation sites, man page alternatives, or API reference.

## Quick Start

The simplest way to generate documentation is with the built-in `--aesh-doc` flag:

```bash
# Generate AsciiDoc to stdout (default)
$ myapp --aesh-doc

# Generate Markdown
$ myapp --aesh-doc markdown

# Save to a file
$ myapp --aesh-doc asciidoc > docs/myapp.adoc
```

No code changes needed — this works automatically with the standard `AeshRuntimeRunner` pattern.

## Programmatic API

For build-time generation or more control, use the `DocumentationGenerator` builder:

```java
import org.aesh.util.doc.DocumentationGenerator;
import org.aesh.util.doc.DocFormat;

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
| **SYNOPSIS** | Formatted usage with options, arguments, and `[COMMAND]` placeholder for groups |
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

## Multi-line Descriptions

Descriptions containing newlines are rendered properly in both `--help` output and generated documentation. This includes text blocks (Java 15+):

```java
@Option(name = "env", description = """
    Target environment.
    Must be one of: dev, staging, prod.
    Defaults to the value of $APP_ENV.""")
```

Leading indentation from text blocks is automatically stripped.
