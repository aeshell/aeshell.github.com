---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Key Bindings'
---

Æsh Readline supports customizable key bindings for editing operations.

## Default Key Bindings

### Emacs Mode

| Action | Key Binding |
|---------|-------------|
| Move back one character | `C-b`, `←` |
| Move forward one character | `C-f`, `→` |
| Delete char left of cursor | `Backspace` |
| Delete char at cursor | `C-d` |
| Undo | `C-_`, `C-x C-u` |
| Move to start of line | `C-a`, `Home` |
| Move to end of line | `C-e`, `End` |
| Move forward word | `M-f` |
| Move backward word | `M-b` |
| Previous history | `↑` |
| Next history | `↓` |
| Clear screen | `C-l` |
| Delete next word | `M-d` |
| Complete | `Tab` |
| Kill to end of line | `C-k` |
| Kill to next word | `M-d` |
| Kill to prev whitespace | `C-w` |
| Yank (paste) | `C-y` |
| Search history backward | `C-r` |
| Search history forward | `C-s` |
| Switch to Vi mode | `M-C-j` |

### Vi Command Mode

| Action | Key Binding |
|---------|-------------|
| Move back one char | `h` |
| Move forward one char | `l` |
| Delete char left of cursor | `X` |
| Delete char at cursor | `x` |
| Undo | `u` |
| Move to start of line | `0` |
| Move to end of line | `$` |
| Move forward word | `w` |
| Move backward word | `b` |
| Previous line | `k`, `↑` |
| Next line | `n`, `↓` |
| Clear screen | `C-l` |
| Delete next word | `dw` |
| Kill to end of line | `D`, `d$` |
| Kill to prev word | `db`, `dB` |
| Yank after cursor | `p` |
| Yank before cursor | `P` |
| Enable change mode | `c` |
| Repeat previous action | `.` |

### Vi Edit Mode

| Action | Key Binding |
|---------|-------------|
| Search history backward | `C-r` |
| Search history forward | `C-s` |
| Delete char left of cursor | `Backspace` |

## Custom Key Bindings

To customize key bindings, implement custom actions and mappings.

## Key Representation

- `C-x` = Ctrl+x
- `M-x` = Meta/Alt+x
- Special keys: `Enter`, `Tab`, `Escape`, `Space`

## Notation Key

- **C** = Control key
- **M** = Meta/Alt key
- Keys are case-sensitive when combined with Control/Meta
