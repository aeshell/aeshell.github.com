---
date: '2026-03-09T16:00:00+01:00'
draft: false
title: 'Graph Display'
weight: 22
---

The graph display utility renders directed acyclic graph (DAG) data as formatted text in the terminal. Unlike the [Tree Display]({{< relref "tree" >}}) where each node has one parent, a DAG allows nodes to have multiple parents (fan-in). Shared dependencies are shown once rather than duplicated, making it ideal for visualizing build pipelines, dependency graphs, and task workflows.

## Quick Start

```java
import org.aesh.util.graph.Graph;
import org.aesh.util.graph.GraphNode;

GraphNode shared = GraphNode.of("C");
GraphNode root = GraphNode.of("Root")
        .child(GraphNode.of("A").child(shared))
        .child(GraphNode.of("B").child(shared));

String output = Graph.render(root);
invocation.println(output);
```

Output (diamond pattern — C is shared between A and B):

```
Root
┌─┴┐
A  B
└┬─┘
 C
```

## GraphNode API

`GraphNode` provides a fluent API for building graph structures. The DAG semantics come from sharing the same node instance across multiple parents:

```java
GraphNode shared = GraphNode.of("shared");
GraphNode root = GraphNode.of("root")
        .child(GraphNode.of("left").child(shared))
        .child(GraphNode.of("right").child(shared));
```

| Method               | Returns     | Description                        |
|----------------------|-------------|------------------------------------|
| `GraphNode.of(label)` | `GraphNode` | Creates a new node                 |
| `child(GraphNode)`    | `this`      | Adds an existing node as child     |
| `child(String)`       | `this`      | Creates and adds a leaf node       |
| `label()`             | `String`    | Returns the node's label           |
| `children()`          | `List`      | Returns unmodifiable children list |

## Static API

For quick rendering of `GraphNode` graphs:

```java
// Default UNICODE style
String output = Graph.render(root);

// Custom style
String output = Graph.render(root, GraphStyle.ASCII);

// With max width (wraps wide layers)
String output = Graph.render(root, 25);

// Custom style + max width
String output = Graph.render(root, GraphStyle.ASCII, 25);

// Custom style + max width + label wrapping
String output = Graph.render(root, GraphStyle.UNICODE, 60, 20);
```

## Builder API

For rendering existing typed DAGs without converting to `GraphNode`:

```java
import org.aesh.util.graph.Graph;
import org.aesh.util.graph.GraphStyle;

String output = Graph.<Task>builder()
        .label(Task::getName)
        .children(Task::getDependencies)
        .style(GraphStyle.UNICODE)
        .maxWidth(40)
        .build()
        .render(rootTask);
```

| Method                           | Default    | Description                           |
|----------------------------------|------------|---------------------------------------|
| `label(Function<T, String>)`     | *required* | Extracts display text from each node  |
| `children(Function<T, List<T>>)` | *required* | Extracts children (null/empty = leaf) |
| `style(GraphStyle)`              | `UNICODE`  | Visual style for connectors           |
| `maxWidth(int)`                  | `0`        | Max output width (0 = no limit)       |
| `maxLabelWidth(int)`             | `0`        | Max label width before wrapping (0 = no wrapping) |

Calling `build()` throws `IllegalStateException` if `label` or `children` are not set.

## Graph Styles

Three predefined styles are available via `GraphStyle`:

**UNICODE** (default):
```
Root
┌─┴┐
A  B
└┬─┘
 C
```

**ASCII**:
```
Root
+-++
A  B
++-+
 C
```

**ROUNDED**:
```
Root
╭─┴╮
A  B
╰┬─╯
 C
```

Each style defines eleven box-drawing characters used for edge routing:

| Character     | UNICODE | ASCII | ROUNDED | Purpose                     |
|---------------|---------|-------|---------|-----------------------------|
| horizontal    | `─`     | `-`   | `─`     | Horizontal edge             |
| vertical      | `│`     | `\|`  | `│`     | Vertical edge               |
| downTee       | `┬`     | `+`   | `┬`     | Parent splits down          |
| upTee         | `┴`     | `+`   | `┴`     | Child joins up              |
| cross         | `┼`     | `+`   | `┼`     | Vertical crosses horizontal |
| topLeft       | `┌`     | `+`   | `╭`     | Corner: down+right          |
| topRight      | `┐`     | `+`   | `╮`     | Corner: down+left           |
| bottomLeft    | `└`     | `+`   | `╰`     | Corner: up+right            |
| bottomRight   | `┘`     | `+`   | `╯`     | Corner: up+left             |
| rightTee      | `├`     | `+`   | `├`     | T-junction: up+down+right   |
| leftTee       | `┤`     | `+`   | `┤`     | T-junction: up+down+left    |

## Common Patterns

### Fan-out (one parent, multiple children)

```java
GraphNode root = GraphNode.of("Root")
        .child("A")
        .child("B")
        .child("C");
```

```
 Root
┌──┼──┐
A  B  C
```

### Diamond (shared dependency)

```java
GraphNode shared = GraphNode.of("C");
GraphNode root = GraphNode.of("Root")
        .child(GraphNode.of("A").child(shared))
        .child(GraphNode.of("B").child(shared));
```

```
Root
┌─┴┐
A  B
└┬─┘
 C
```

### Complex DAG (multiple shared nodes)

```java
GraphNode d = GraphNode.of("D");
GraphNode a = GraphNode.of("A").child("C").child(d);
GraphNode b = GraphNode.of("B").child(d).child("E");
GraphNode root = GraphNode.of("Root").child(a).child(b);
```

```
 Root
 ┌─┴┐
 A  B
┌┴─┐│
│  ├┴─┐
C  D  E
```

Here D is shared between A and B — it appears once with edges from both parents. Each parent's edges are drawn on separate routing rows to avoid visual ambiguity.

## Width Limiting

When a node has many children, the graph can grow very wide. The `maxWidth` parameter constrains the output by wrapping wide layers onto multiple rows. Children that don't fit are moved to additional rows, with vertical connectors routing edges through the intermediate layers automatically.

```java
GraphNode root = GraphNode.of("Pipeline")
        .child("Compile")
        .child("Test")
        .child("Lint")
        .child("Package")
        .child("Deploy")
        .child("Notify");

// Without maxWidth — all children on one wide row:
Graph.render(root);
```

```
                  Pipeline
   ┌───────┬─────┬────┴─┬────────┬───────┐
Compile  Test  Lint  Package  Deploy  Notify
```

```java
// With maxWidth=25 — children wrap to fit:
Graph.render(root, 25);
```

```
          Pipeline
   ┌─────┬──┬─┴──┬───┬────┐
Compile  │  │  Test  │  Lint
     ┌───┘  │        │
     │      └─┐      │
     │        │      └┐
  Package  Deploy  Notify
```

A `maxWidth` of `0` (the default) means no limit. The wrapping applies to the node label layers; routing lines between layers may extend slightly beyond `maxWidth` when connecting nodes across split rows.

## Label Wrapping

When node labels are long, the `maxLabelWidth` parameter wraps them at word boundaries. Each label that exceeds the limit is split into multiple lines, and the graph layout accounts for the variable node heights.

```java
GraphNode root = GraphNode.of("Build and Deploy Pipeline")
        .child("Compile Source Code")
        .child("Run Integration Tests");

// Without label wrapping:
Graph.render(root);
```

```
     Build and Deploy Pipeline
     ┌──────────────┴──────────┐
Compile Source Code   Run Integration Tests
```

```java
// With maxLabelWidth=12:
Graph.render(root, GraphStyle.UNICODE, 0, 12);
```

```
  Build and
   Deploy
  Pipeline
  ┌───┴────┐
Compile  Run
Source   Integration
 Code    Tests
```

Labels wrap at spaces when possible. If a label has no spaces within the limit, it hard-breaks at the exact character position.

With the builder API:

```java
String output = Graph.<Task>builder()
        .label(Task::getDescription)
        .children(Task::getDependencies)
        .maxLabelWidth(20)
        .build()
        .render(rootTask);
```

A `maxLabelWidth` of `0` (the default) means no wrapping.

## Using in Commands

Here is a complete command example that displays a build dependency graph:

```java
@CommandDefinition(name = "deps", description = "Display dependency graph")
public class DepsCommand implements Command<CommandInvocation> {

    @Option(name = "style", shortName = 's', defaultValue = {"UNICODE"},
            description = "Graph style: ASCII, UNICODE, ROUNDED")
    private GraphStyle style;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Build a dependency graph
        GraphNode test = GraphNode.of("test");
        GraphNode compile = GraphNode.of("compile");
        GraphNode lint = GraphNode.of("lint");
        GraphNode validate = GraphNode.of("validate").child(compile).child(lint);
        GraphNode root = GraphNode.of("build")
                .child(validate)
                .child(test);

        String output = Graph.render(root, style);
        invocation.println(output);
        return CommandResult.SUCCESS;
    }
}
```

## Cycle Detection

Graphs must be acyclic. If the graph contains a cycle, `render()` throws an `IllegalArgumentException`:

```java
GraphNode a = GraphNode.of("A");
GraphNode b = GraphNode.of("B").child(a);
a.child(b); // creates A → B → A cycle

Graph.render(a); // throws IllegalArgumentException: "Graph contains a cycle"
```

## Edge Cases

| Scenario                        | Behavior                                       |
|---------------------------------|------------------------------------------------|
| Root with no children           | Only the root label is printed                 |
| `children()` returns `null`     | Treated as a leaf node (no crash)              |
| `children()` returns empty list | Treated as a leaf node                         |
| Cyclic graph                    | Throws `IllegalArgumentException`              |
| Shared child node               | Rendered once with edges from all parents      |

## Tree vs Graph

| Feature            | Tree                      | Graph                       |
|--------------------|---------------------------|-----------------------------|
| Structure          | Each node has one parent  | Nodes can have many parents |
| Rendering          | Vertical with indentation | Layered horizontal layout   |
| Shared nodes       | Duplicated in output      | Shown once with fan-in      |
| Cycle detection    | N/A (tree structure)      | Throws on cycles            |
| Use case           | File trees, hierarchies   | Dependencies, pipelines     |

Use `Tree` for simple hierarchies (file systems, org charts). Use `Graph` when nodes can be shared across multiple parents (build steps, dependency resolution).

## API Reference

### GraphNode

| Method               | Returns     | Description                    |
|----------------------|-------------|--------------------------------|
| `of(String)`         | `GraphNode` | Factory method                 |
| `child(GraphNode)`   | `GraphNode` | Add child node, returns this   |
| `child(String)`      | `GraphNode` | Add leaf child, returns this   |
| `label()`            | `String`    | Get label                      |
| `children()`         | `List`      | Unmodifiable children list     |

### Graph (static)

| Method                                    | Returns    | Description                              |
|-------------------------------------------|------------|------------------------------------------|
| `render(GraphNode)`                       | `String`   | Render with UNICODE style                |
| `render(GraphNode, int)`                  | `String`   | Render with UNICODE style + max width    |
| `render(GraphNode, GraphStyle)`           | `String`   | Render with specified style              |
| `render(GraphNode, GraphStyle, int)`      | `String`   | Render with specified style + max width  |
| `render(GraphNode, GraphStyle, int, int)` | `String`   | Render with style + max width + label wrapping |
| `builder()`                               | `Builder`  | Create a generic builder                 |

### GraphStyle

| Constant  | Description                             |
|-----------|-----------------------------------------|
| `ASCII`   | Uses `+`, `-`, `\|` characters          |
| `UNICODE` | Uses box-drawing characters (default)   |
| `ROUNDED` | Uses rounded corners (`╭`, `╮`, `╰`, `╯`) |

## See Also

- [Tree Display]({{< relref "tree" >}}) — Hierarchical tree rendering
- [Table Display]({{< relref "table" >}}) — Tabular data rendering
- [Progress Bar]({{< relref "progress-bar" >}}) — Progress feedback for long-running operations
