---
date: '2026-02-11T16:00:00+01:00'
draft: false
title: 'Tree Display'
weight: 21
---

The tree display utility renders hierarchical data as formatted text trees in the terminal. It supports multiple visual styles, depth limiting, and both a simple node-based API and a generic builder for traversing existing typed hierarchies.

## Quick Start

```java
import org.aesh.util.tree.Tree;
import org.aesh.util.tree.TreeNode;

TreeNode root = TreeNode.of("project")
        .child(TreeNode.of("src")
                .child("Main.java")
                .child("Util.java"))
        .child(TreeNode.of("test")
                .child("MainTest.java"));

String output = Tree.render(root);
invocation.println(output);
```

Output:

```
project
├── src
│   ├── Main.java
│   └── Util.java
└── test
    └── MainTest.java
```

## TreeNode API

`TreeNode` provides a fluent API for building tree structures manually:

```java
TreeNode root = TreeNode.of("root")          // factory method
        .child(TreeNode.of("branch")         // add subtree
                .child("leaf1")              // add leaf by label
                .child("leaf2"))
        .child("another-leaf");
```

| Method              | Returns    | Description                      |
|---------------------|------------|----------------------------------|
| `TreeNode.of(label)` | `TreeNode` | Creates a new node               |
| `child(TreeNode)`    | `this`     | Adds an existing node as child   |
| `child(String)`      | `this`     | Creates and adds a leaf node     |
| `label()`            | `String`   | Returns the node's label         |
| `children()`         | `List`     | Returns unmodifiable children list |

## Static API

For quick rendering of `TreeNode` trees:

```java
// Default UNICODE style
String output = Tree.render(root);

// Custom style
String output = Tree.render(root, TreeStyle.ASCII);
```

## Builder API

For rendering existing typed hierarchies without converting to `TreeNode`:

```java
import org.aesh.util.tree.Tree;
import org.aesh.util.tree.TreeStyle;

String output = Tree.<File>builder()
        .label(File::getName)
        .children(f -> f.isDirectory()
                ? Arrays.asList(f.listFiles())
                : Collections.emptyList())
        .style(TreeStyle.UNICODE)
        .maxDepth(3)
        .build()
        .render(rootDir);
```

| Method                             | Default    | Description                                     |
|------------------------------------|------------|-------------------------------------------------|
| `label(Function<T, String>)`       | *required* | Extracts display text from each node             |
| `children(Function<T, List<T>>)`   | *required* | Extracts children (null/empty = leaf)            |
| `style(TreeStyle)`                 | `UNICODE`  | Visual style for connectors                      |
| `maxDepth(int)`                    | `-1`       | Max depth (-1 = unlimited, 0 = root's children)  |

Calling `build()` throws `IllegalStateException` if `label` or `children` are not set.

## Tree Styles

Four predefined styles are available via `TreeStyle`:

**UNICODE** (default):
```
root
├── child1
│   ├── grandchild1
│   └── grandchild2
└── child2
```

**ASCII**:
```
root
+-- child1
|   +-- grandchild1
|   \-- grandchild2
\-- child2
```

**COMPACT**:
```
root
├─ child1
│  ├─ grandchild1
│  └─ grandchild2
└─ child2
```

**ROUNDED**:
```
root
├── child1
│   ├── grandchild1
│   ╰── grandchild2
╰── child2
```

## Max Depth

Use `maxDepth` to limit how deep the tree renders. When a node has children beyond the depth limit, `...` is displayed:

```java
String output = Tree.<TreeNode>builder()
        .label(TreeNode::label)
        .children(TreeNode::children)
        .maxDepth(1)
        .build()
        .render(root);
```

```
root
├── child1
│   ├── grandchild1
│   └── grandchild2
└── child2
    ...
```

With `maxDepth(0)`, only the root's direct children are shown:

```
root
├── child1
│   ...
└── child2
    ...
```

## Using in Commands

Here is a complete command example that displays a directory tree:

```java
@CommandDefinition(name = "tree", description = "Display directory tree")
public class TreeCommand implements Command<CommandInvocation> {

    @Argument(description = "Root directory")
    private Resource root;

    @Option(name = "depth", shortName = 'd', defaultValue = {"-1"},
            description = "Max depth (-1 for unlimited)")
    private int maxDepth;

    @Option(name = "style", shortName = 's', defaultValue = {"UNICODE"},
            description = "Tree style: ASCII, UNICODE, COMPACT, ROUNDED")
    private TreeStyle style;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        File dir = new File(root.getAbsolutePath());
        String output = Tree.<File>builder()
                .label(File::getName)
                .children(f -> f.isDirectory()
                        ? Arrays.asList(f.listFiles())
                        : Collections.emptyList())
                .style(style)
                .maxDepth(maxDepth)
                .build()
                .render(dir);

        invocation.println(output);
        return CommandResult.SUCCESS;
    }
}
```

## Edge Cases

| Scenario                           | Behavior                              |
|------------------------------------|---------------------------------------|
| Root with no children              | Only the root label is printed        |
| `children()` returns `null`        | Treated as a leaf node (no crash)     |
| `children()` returns empty list    | Treated as a leaf node                |
| `maxDepth(-1)`                     | Unlimited depth (default)             |
| `maxDepth(0)`                      | Root's children only, truncated below |

## API Reference

### TreeNode

| Method              | Returns    | Description                      |
|---------------------|------------|----------------------------------|
| `of(String)`        | `TreeNode` | Factory method                   |
| `child(TreeNode)`   | `TreeNode` | Add subtree, returns this        |
| `child(String)`     | `TreeNode` | Add leaf, returns this           |
| `label()`           | `String`   | Get label                        |
| `children()`        | `List`     | Unmodifiable children list       |

### Tree (static)

| Method                        | Returns  | Description                    |
|-------------------------------|----------|--------------------------------|
| `render(TreeNode)`            | `String` | Render with UNICODE style      |
| `render(TreeNode, TreeStyle)` | `String` | Render with specified style    |
| `builder()`                   | `Builder`| Create a generic builder       |

### TreeStyle

| Constant  | Branch | Last   | Vertical | Space  |
|-----------|--------|--------|----------|--------|
| `ASCII`   | `+-- ` | `\-- ` | `\|   `  | `    ` |
| `UNICODE` | `├── ` | `└── ` | `│   `   | `    ` |
| `COMPACT` | `├─ `  | `└─ `  | `│  `    | `   `  |
| `ROUNDED` | `├── ` | `╰── ` | `│   `   | `    ` |

## See Also

- [Graph Display]({{< relref "graph" >}}) — DAG rendering for shared dependencies
- [Table Display]({{< relref "table" >}}) — Tabular data rendering
- [Progress Bar]({{< relref "progress-bar" >}}) — Progress feedback for long-running operations
