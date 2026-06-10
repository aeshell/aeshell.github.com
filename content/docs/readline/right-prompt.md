---
date: '2026-06-10T15:00:00+01:00'
draft: false
title: 'Right Prompt'
weight: 12
---

Display a right-aligned prompt on the prompt line that automatically disappears when the input grows too long.

## Usage

```java
Prompt prompt = Prompt.builder()
    .message("$ ")
    .rightPrompt("git:main")
    .build();

readline.readline(connection, prompt, line -> { ... });
```

## Behavior

- The right prompt is displayed right-aligned on the first line of the prompt
- When the user's input grows long enough to overlap with the right prompt, it automatically hides
- When the input shrinks (e.g., backspace), the right prompt reappears
- If the input wraps to multiple lines, the right prompt is not shown

## Dynamic Right Prompt

The right prompt can be updated between readline sessions:

```java
// Show current time as right prompt
prompt.setRightPrompt(LocalTime.now().format(DateTimeFormatter.ofPattern("HH:mm")));
```

## ANSI Support

The right prompt can contain ANSI escape sequences:

```java
Prompt prompt = Prompt.builder()
    .message("$ ")
    .rightPrompt("\033[36mgit:main\033[0m")
    .build();
```

## Example

See `RightPromptExample` in the examples directory.
