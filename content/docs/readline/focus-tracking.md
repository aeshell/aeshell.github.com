---
date: '2026-06-10T15:00:00+01:00'
draft: false
title: 'Focus Tracking'
weight: 22
---

Detect when the terminal window gains or loses focus. Useful for pausing animations, dimming the UI, or triggering refreshes when the user returns.

## Usage

```java
// Enable focus tracking with a handler
connection.terminal().enableFocusTracking(focused -> {
    if (focused) {
        connection.write("Welcome back!\n");
    } else {
        connection.write("See you soon...\n");
    }
});

// Disable when done
connection.terminal().disableFocusTracking();
```

## How It Works

Focus tracking uses the terminal's `ESC [ ? 1004 h` mode:
- `ESC [ I` is sent when the terminal gains focus
- `ESC [ O` is sent when the terminal loses focus

The `EventDecoder` uses the VtParser-based sequence filter to intercept these sequences and route them to the focus handler, preventing them from appearing as input. The filter correctly handles sequences split across input buffer boundaries -- if `ESC` arrives in one read and `[ I` in the next, the focus event is still detected.

## With Readline

The focus handler works alongside readline — focus events are intercepted before reaching the input handler.

## tmux Note

If running inside tmux, focus events require `set -g focus-events on` in `tmux.conf`.

## Example

See `FocusTrackingExample` in the examples directory.
