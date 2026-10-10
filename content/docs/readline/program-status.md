---
date: '2026-10-10T12:00:00+01:00'
draft: false
title: 'Program Status'
weight: 18
---

Æsh Readline supports the OSC 7501 Program Status Protocol ([specification](https://www.superlogical.com/rex/docs/build/program-status), revision 0.3). A program reports what it is doing — working, blocked, done — and the terminal decides how to present it: a tab indicator, an inbox entry, a notification, or nothing at all. Terminals that do not recognize OSC 7501 ignore the sequence, so reporting is safe everywhere with no prior negotiation.

## How It Works

One escape sequence carries a report as `key=value` pairs:

```
ESC ] 7501 ; state=working:app=deploy:msg=RGVwbG95aW5nIHYyLjQuMQ== ESC \
```

Free-text values (`msg`, `title`) are base64-encoded UTF-8. Every report completely replaces its addressed record, so a caller repeats metadata it wants retained, including `app` and `title`.

| State | Meaning |
|-------|---------|
| `idle` | At rest, waiting for the next instruction |
| `working` | Running; may carry a `progress` percentage |
| `done` | Finished; the result is ready and not yet seen |
| `blocked` | Cannot continue until the user acts; `kind` says what (`permission`, `question`, `auth`) |
| `error` | Failed and stopped |
| `clear` | Not a state; removes the addressed record (or every record with no id) |

`done` and `error` records deliberately survive prompts and process exit until replaced or cleared. Never overwrite them with `idle` at every prompt.

Programs doing several things at once address child records with hierarchical ids (`deploy/eu-west`). Each record holds exactly the keys of its last report.

## Quick Start

```java
import org.aesh.terminal.TerminalFeatures;
import org.aesh.terminal.utils.ProgramStatus;

TerminalFeatures terminal = connection.terminal();

// Report blocking work with a plain-text message
terminal.writeProgramStatus(
    ProgramStatus.builder(ProgramStatus.State.BLOCKED)
        .app("deploy")
        .id("deploy/eu-west")
        .title("EU West")
        .kind(ProgramStatus.BlockedKind.PERMISSION)
        .message("Approve deployment?")
        .build());

// Back to work with progress
terminal.writeProgramStatus(
    ProgramStatus.builder(ProgramStatus.State.WORKING)
        .id("deploy/eu-west")
        .title("EU West")
        .progress(40)
        .message("Pushing image")
        .build());

// Done, then remove the child record
terminal.writeProgramStatus(
    ProgramStatus.builder(ProgramStatus.State.DONE)
        .id("deploy/eu-west")
        .title("EU West")
        .message("Healthy")
        .build());
terminal.clearProgramStatus("deploy/eu-west");
```

Developers pass plain text and typed values only. UTF-8 encoding, base64, validation, and sequence framing stay inside the implementation, which rejects invalid reports instead of truncating them. Each write sends one complete sequence through the existing `Connection` output, so reports work over every transport and never require a support query first.

## Checking Support

Detection is optional but available when a program wants to adapt:

```java
// Terminfo hint only; a missing entry never means unsupported
boolean hinted = terminal.hasProgramStatusHint();

// Live query through the existing connection reader.
// True/false is cached per connection; null means unknown (timeout).
Boolean supported = terminal.queryProgramStatusSupport(500);
```

The live query sends an `OSC 7501 ; ?` probe followed by a device-attributes query as a fence: a probe reply first means supported, the fence reply first means unsupported. Unrelated keyboard input is preserved, and conclusive answers are cached on the connection while timeouts retry on the next call.

## Terminal Support

| Terminal | Supported |
|----------|-----------|
| Ghostty | Yes |
| Rex (Superlogical) | Yes |
| Others | Unknown sequences are ignored; reporting is safe but not displayed |

Supporting OSC 7501 does not guarantee any particular presentation such as a desktop notification. Keep any notification policy independently configurable.

## Connection API

Status methods are on the `TerminalFeatures` delegate, accessed via `connection.terminal()`:

| Method | Purpose |
|--------|---------|
| `writeProgramStatus(status)` | Send one validated report; never probes or reads |
| `clearProgramStatus(id)` | Remove one record and everything beneath it |
| `clearAllProgramStatus()` | Remove every record, including other programs' |
| `hasProgramStatusHint()` | Check the terminfo `Pst` hint |
| `queryProgramStatusSupport(timeoutMs)` | Live probe with per-connection cache |
| `programStatusSupport()` | Read the cached answer (true/false/null) |

There is no automatic command-state inference in readline and no receiver-side record store. Applications and frameworks like aesh decide when work starts, blocks, completes, or fails, and report it explicitly.

## See Also

- [Connection]({{< relref "connection" >}}) — the `TerminalFeatures` delegate
- [Hyperlinks]({{< relref "hyperlinks" >}}) — another OSC-based semantic output
- [Clipboard]({{< relref "clipboard" >}}) — explicit terminal output without queries
