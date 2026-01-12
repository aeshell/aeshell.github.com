---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Edit Modes'
weight: 4
---

Æsh Readline supports two main editing modes: Emacs and Vi.

## EditMode

```java
import org.aesh.readline.editing.EditMode;
import org.aesh.readline.editing.EditModeBuilder;

// Create default edit mode (Emacs)
EditMode editMode = EditModeBuilder.builder().create();

// Create Vi mode
EditMode viMode = EditModeBuilder.builder()
        .mode(EditMode.Mode.VI)
        .create();
```

## Emacs Mode

Default mode with Emacs-style key bindings.

### Setting Emacs Mode

```java
import org.aesh.readline.ReadlineBuilder;

Readline readline = ReadlineBuilder.builder()
        .editMode(EditModeBuilder.builder()
                .mode(EditMode.Mode.EMACS)
                .create())
        .build();
```

### Emacs Key Bindings

| Key | Action |
|-----|--------|
| `C-b` or `←` | Move back one character |
| `C-f` or `→` | Move forward one character |
| `Backspace` | Delete character left of cursor |
| `C-d` | Delete character at cursor |
| `C-_` or `C-x C-u` | Undo |
| `C-a` or `Home` | Move to start of line |
| `C-e` or `End` | Move to end of line |
| `M-f` | Move forward one word |
| `M-b` | Move backward one word |
| `↑` | Previous history |
| `↓` | Next history |
| `C-l` | Clear screen, redraw line |
| `M-d` | Delete next word |
| `C-k` | Kill to end of line |
| `C-w` | Kill to previous whitespace |
| `C-y` | Yank (paste) |
| `C-r` | Search backward in history |
| `C-s` | Search forward in history |
| `M-C-j` | Switch to Vi mode |
| `Tab` | Complete |

## Vi Mode

Vi-style editing with command and insert modes.

### Setting Vi Mode

```java
import org.aesh.readline.ReadlineBuilder;

Readline readline = ReadlineBuilder.builder()
        .editMode(EditModeBuilder.builder()
                .mode(EditMode.Mode.VI)
                .create())
        .build();
```

### Vi Command Mode

| Key | Action |
|-----|--------|
| `h` | Move back one character |
| `l` | Move forward one character |
| `X` | Delete character left of cursor |
| `x` | Delete character at cursor |
| `u` | Undo |
| `0` | Move to start of line |
| `$` | Move to end of line |
| `w` | Move forward one word |
| `b` | Move backward one word |
| `k` or `↑` | Previous line |
| `n` or `↓` | Next line |
| `C-l` | Clear screen |
| `dw` | Delete next word |
| `D` or `d$` | Kill to end of line |
| `db` | Kill to previous whitespace |
| `p` | Yank after cursor |
| `P` | Yank before cursor |
| `y` + movement | Add text to yank buffer |
| `c` | Enable change mode |
| `.` | Repeat previous action |

### Vi Edit Mode

| Key | Action |
|-----|--------|
| `C-r` | Search backward in history |
| `C-s` | Search forward in history |
| `Backspace` | Delete character left of cursor |
| `Esc` | Return to command mode |

## Switching Modes

Users can switch between modes at runtime:

- In Emacs mode: Press `M-C-j` (Alt+Ctrl+j) to switch to Vi mode
- In Vi mode: Press `Esc` to exit to command mode, then type editing commands

## Runtime Properties

Set edit mode via system property:

```java
System.setProperty("aesh.editmode", "VI");
// Or
System.setProperty("aesh.editmode", "EMACS");
```
