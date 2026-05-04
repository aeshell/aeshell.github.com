---
date: '2026-04-29T14:00:00+02:00'
draft: false
title: 'File and Resource Handling'
weight: 12
---

When an `@Option`, `@Argument`, or `@Arguments` field is typed as `Resource`, `File`, or `Path`, Aesh automatically provides file path completion, string-to-file conversion, and `<file>` usage hints in help text -- with no extra configuration needed.

## Quick Comparison

```java
// Plain string — no completion, no path resolution
@Option(description = "Output path")
private String output;

// Resource — automatic file completion + path resolution + I/O API
@Option(description = "Output path")
private Resource output;

// File — automatic file completion + conversion
@Option(description = "Output path")
private File output;

// Path — automatic file completion + conversion
@Option(description = "Output path")
private Path output;
```

All three file-typed variants (`Resource`, `File`, `Path`) gain:

| Feature | String | Resource / File / Path |
|---|---|---|
| Tab completion | None | File paths from filesystem |
| Type conversion | None | Automatic string-to-type |
| Help text | `<output>` | `<file>` |
| Shell redirect completion | No | Yes (after `>`, `>>`, `<`) |

## Resource Interface

`Resource` is Aesh's filesystem abstraction. It provides a richer API than `File` or `Path` with built-in glob expansion, filtering, and I/O streams:

```java
@CommandDefinition(name = "cat", description = "Display file contents")
public class CatCommand implements Command<CommandInvocation> {

    @Argument(description = "File to display", required = true)
    private Resource file;

    @Override
    public CommandResult execute(CommandInvocation invocation) throws CommandException {
        if (!file.exists()) {
            invocation.println("File not found: " + file.getName());
            return CommandResult.FAILURE;
        }
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(file.read()))) {
            String line;
            while ((line = reader.readLine()) != null) {
                invocation.println(line);
            }
        } catch (IOException e) {
            throw new CommandException("Failed to read file", e);
        }
        return CommandResult.SUCCESS;
    }
}
```

Tab completion works immediately:

```
myshell$ cat sr<TAB>
src/
myshell$ cat src/main<TAB>
src/main/java/  src/main/resources/
```

### API Reference

**Path and name:**

| Method | Returns | Description |
|---|---|---|
| `getName()` | `String` | Filename only (no directory) |
| `getAbsolutePath()` | `String` | Full absolute path |
| `getParent()` | `Resource` | Parent directory |
| `newInstance(String)` | `Resource` | Factory -- creates a new Resource from a path |

**Type checks:**

| Method | Returns | Description |
|---|---|---|
| `exists()` | `boolean` | Does the file/directory exist? |
| `isLeaf()` | `boolean` | Is it a file (not a directory)? |
| `isDirectory()` | `boolean` | Is it a directory? |
| `isSymbolicLink()` | `boolean` | Is it a symbolic link? |
| `readSymbolicLink()` | `Resource` | Resolves symlink target |

**Directory listing:**

| Method | Returns | Description |
|---|---|---|
| `list()` | `List<Resource>` | Lists directory contents |
| `list(ResourceFilter)` | `List<Resource>` | Lists with filtering |
| `listRoots()` | `List<Resource>` | Filesystem roots |
| `resolve(Resource cwd)` | `List<Resource>` | Resolves path with glob expansion (`~`, `*`, `?`) |

**I/O:**

| Method | Returns | Description |
|---|---|---|
| `read()` | `InputStream` | Opens file for reading |
| `write(boolean append)` | `OutputStream` | Opens file for writing (append or overwrite) |

**File operations:**

| Method | Returns | Description |
|---|---|---|
| `mkdirs()` | `boolean` | Creates directory and parents |
| `delete()` | `boolean` | Deletes file or empty directory |
| `move(Resource)` | `void` | Moves/renames file |
| `copy(Resource)` | `Resource` | Copies file or directory |

**Metadata:**

| Method | Returns | Description |
|---|---|---|
| `lastModified()` | `long` | Last modified time (ms) |
| `setLastModified(long)` | `boolean` | Sets last modified time |
| `lastAccessed()` | `long` | Last accessed time (ms) |

## FileResource

`FileResource` is the default `Resource` implementation backed by `java.io.File`. When a command field is typed as `Resource`, the input string is automatically converted to a `FileResource` resolved relative to the current working directory.

```java
@Option(description = "Config file")
private Resource config;

// In execute():
if (config.isLeaf() && config.exists()) {
    // read the file
    InputStream in = config.read();
}

// Access the underlying File if needed:
File javaFile = ((FileResource) config).getFile();
```

You can also construct `FileResource` directly:

```java
Resource home = new FileResource(System.getProperty("user.home"));
Resource config = new FileResource("/etc/myapp/config.yml");
Resource relative = new FileResource("data/output.csv");
```

## Resource Filters

Resource filters control which files appear in directory listings and tab completion. Four built-in filters are available:

| Filter | Accepts |
|---|---|
| `AllResourceFilter` | Everything (default) |
| `DirectoryResourceFilter` | Directories only |
| `LeafResourceFilter` | Files only (not directories) |
| `NoDotNamesFilter` | Files not starting with `.` |

### Filtering Completion Candidates

To restrict tab completion to directories only, provide a custom completer using `FileOptionCompleter` with a filter:

```java
@CommandDefinition(name = "cd", description = "Change directory")
public class CdCommand implements Command<CommandInvocation> {

    @Argument(description = "Target directory",
              completer = DirectoryCompleter.class)
    private Resource target;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (target != null && target.isDirectory()) {
            invocation.getAeshContext().setCurrentWorkingDirectory(target);
        }
        return CommandResult.SUCCESS;
    }
}

public class DirectoryCompleter extends FileOptionCompleter {
    public DirectoryCompleter() {
        super(new DirectoryResourceFilter());
    }
}
```

Now tab completion only suggests directories:

```
myshell$ cd sr<TAB>
src/
myshell$ cd src/<TAB>
src/main/  src/test/
```

### Filtering Directory Listings

Filters work with `Resource.list()` too:

```java
// List only non-hidden files
List<Resource> visible = directory.list(new NoDotNamesFilter());

// List only subdirectories
List<Resource> dirs = directory.list(new DirectoryResourceFilter());
```

### Custom Filters

Implement `ResourceFilter` for custom logic:

```java
public class JavaFileFilter implements ResourceFilter {
    @Override
    public boolean accept(Resource resource) {
        return resource.isDirectory() || resource.getName().endsWith(".java");
    }
}
```

## Glob Expansion

`Resource.resolve()` supports shell-style globs:

```java
Resource cwd = invocation.getAeshContext().getCurrentWorkingDirectory();
Resource pattern = new FileResource("src/**/*.java");
List<Resource> matches = pattern.resolve(cwd);
```

The `~` character is expanded to the user's home directory:

```java
Resource home = new FileResource("~/documents");
List<Resource> resolved = home.resolve(cwd);
```

## Resource vs File vs Path

| | `Resource` | `File` | `Path` |
|---|---|---|---|
| **Tab completion** | Automatic | Automatic | Automatic |
| **API richness** | Full (listing, glob, I/O, copy/move) | Java standard | Java NIO |
| **Glob expansion** | Built-in via `resolve()` | Manual | Via `PathMatcher` |
| **Filtering** | `ResourceFilter` integration | Manual | Manual |
| **Abstraction** | Pluggable (file, pipeline, remote) | Local filesystem only | Local filesystem only |
| **Recommendation** | Use for commands that work with files | Use when you need `java.io.File` APIs | Use when you need `java.nio.file` APIs |

Use `Resource` when your command needs filesystem operations and you want the richest API. Use `File` or `Path` if you need compatibility with existing Java APIs that expect those types.

## See Also

- [Options](../options) -- Defining command options
- [Arguments](../arguments) -- Positional arguments
- [Completers](../completers) -- Custom tab completion
- [Converters](../converters) -- Built-in and custom type conversion
