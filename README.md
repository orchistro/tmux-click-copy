# tmux_click_copy

> [!NOTE]
> **This fork is retired (November 2026).** I no longer load it. The same behaviour now lives in a few plain key bindings in my `tmux.conf`. See [Why this fork is retired](#why-this-fork-is-retired) at the end of this file. The rest of this README describes the plugin as it was.

This plugin rebinds `DoubleClick1Pane` and `TripleClick1Pane` in the root and copy-mode key-tables with the default (when mouse on) with the addition to not stop the selection upon releasing the mouse and starting an async timer of 0.5/0.7 seconds to stop the selection and exit copy mode if it remains unaltered after the timer ends.

This enables to drag to extend the selection with double or triple click with the 'word-selection' and 'line-selection' way of selecting, but also does not leave the pane in copy mode if using just a double (or triple) click to select just the word or the line without dragging the mouse.

Double & Triple click base taken and modified from [ideasman dotfiles](https://gitlab.com/ideasman42/dotfiles) with the addition of a bash script that after a timeout exits copy-mode if the selection has not changed.

# Usage
Install this plugin via the [tmux plugin manager](https://github.com/tmux-plugins/tpm) adding this to your tmux configuration file:
``` shell
set -g @plugin "aless3/tmux-click-copy"
```
Send `prefix+I` to fetch and install the plugin and source it.
The default timeout after which this plugin copies the selection and exits copy mode is 0.5s for double click selections and 0.7s for triple click selections.

You can modify setting the options `@tcc_timeout` (for double click) and `@tcc_long_timeout` (for triple click) before loading it:
``` shell
set -g @tcc_timeout 10
set -g @tcc_long_timeout 100
set -g @plugin "aless3/tmux-click-copy"
```

This plugin automatically binds `DoubleClick1Pane` and `TripleClick1Pane` in the root and copy-mode key-tables, to disable auto-binding set `@tcc_autobind` to false:
``` shell
set -g @tcc_autobind false
set -g @plugin "aless3/tmux-click-copy"
```

The shell script command used internally is stored also in the global options `@TCC_COMMAND` and `@TCC_COMMAND_TRIPLE`, so you can bind it how you want.

# Why this fork is retired

I stopped using this plugin and replaced it with plain key bindings in `tmux.conf`. The reasons:

1. **tmux already handles double and triple click.** The default `DoubleClick1Pane` and `TripleClick1Pane` bindings select the word or the line, copy it, and leave copy mode 0.3 seconds later. All I wanted on top of that was to stay in copy mode, keep the selection visible, and extend it by word or by line while dragging. That takes a handful of bindings, not a plugin.
2. **The timer script no longer does anything useful in this fork.** Upstream uses the timer to copy the selection and leave copy mode once the selection has stopped changing. This fork was changed to stay in copy mode, so all the timer does now is copy the same selection a second time.
3. **tmux 3.8 broke drag-to-extend, and the fix is a key binding either way.** Since tmux 3.8, a drag that starts inside an existing selection adjusts the nearest end of that selection and resets the selection mode to `char`. So dragging after a double or triple click no longer extends by word or by line: double-clicking `beta` in `alpha beta gamma delta` and dragging to `delta` copies `a gamma del` instead of `beta gamma delta`. The click bindings of this plugin still work. The drag is handled by `MouseDrag1Pane`, which this plugin never bound, so the workaround is a new `MouseDrag1Pane` binding with or without the plugin. This fork does not contain that workaround.

## Replacement bindings

These go in `tmux.conf`. They are written for `mode-keys vi`; with emacs keys, use the `copy-mode` table instead of `copy-mode-vi`.

``` shell
# keep the selection and stay in copy mode when the mouse button is released,
# and start over on a new click.
# when the button is released on the top or bottom row, also stop the
# autoscroll, which would otherwise keep running for as long as copy mode lasts
bind-key -T copy-mode-vi MouseDragEnd1Pane {
  send-keys -X copy-selection-no-clear
  if -F '#{||:#{==:#{copy_cursor_y},0},#{==:#{copy_cursor_y},#{e|-:#{pane_height},1}}}' {
    send-keys -X stop-selection
    if -F '#{==:#{copy_cursor_y},0}' {
      send-keys -X cursor-down
    } {
      send-keys -X cursor-up
    }
  }
}
bind-key -T copy-mode-vi MouseDown1Pane select-pane \; send-keys -X clear-selection

# double click selects a word, triple click selects a line, both stay in copy mode
bind-key -T root SecondClick1Pane {
  select-pane -t =
  if -F '#{||:#{pane_in_mode},#{mouse_any_flag}}' {
    send-keys -M
  } {
    copy-mode -H
    send-keys -X select-word
    send-keys -X copy-pipe-no-clear
  }
}
bind-key -T root TripleClick1Pane {
  select-pane -t =
  if -F '#{||:#{pane_in_mode},#{mouse_any_flag}}' {
    send-keys -M
  } {
    copy-mode -H
    send-keys -X select-line
    send-keys -X copy-pipe-no-clear
  }
}
bind-key -T copy-mode-vi SecondClick1Pane select-pane \; send-keys -X select-word \; send-keys -X copy-pipe-no-clear
bind-key -T copy-mode-vi TripleClick1Pane select-pane \; send-keys -X select-line \; send-keys -X copy-pipe-no-clear

# the default DoubleClick bindings select the word and leave copy mode
# 0.3 seconds later, so make them do nothing
bind-key -T root DoubleClick1Pane select-pane -t =
bind-key -T copy-mode-vi DoubleClick1Pane select-pane

# tmux 3.8 workaround: when begin-selection has reset the selection mode to char,
# select the original word or line again and extend it to the mouse position
bind-key -T copy-mode-vi User900 send-keys -X other-end \; send-keys -X select-word
bind-key -T copy-mode-vi User901 send-keys -X other-end \; send-keys -X select-line
bind-key -T copy-mode-vi MouseDrag1Pane {
  select-pane
  if -F '#{==:#{selection_mode},word}' {
    send-keys -X begin-selection
    if -F '#{==:#{selection_mode},char}' {
      send-keys User900
      send-keys -X rectangle-off
    }
  } {
    if -F '#{==:#{selection_mode},line}' {
      send-keys -X begin-selection
      if -F '#{==:#{selection_mode},char}' {
        send-keys User901
        send-keys -X rectangle-off
      }
    } {
      send-keys -X begin-selection
    }
  }
}
```

Why the bindings look the way they do:

- **`SecondClick1Pane`, not `DoubleClick1Pane`.** tmux sends `DoubleClick1Pane` 0.3 seconds after the second press, once it is sure that no third click follows. A drag that starts within those 0.3 seconds begins before the word is selected, so it extends by character. `SecondClick1Pane` is sent at the second press itself.
- **`MouseDown1Pane`.** The workaround depends on it. `clear-selection` resets the selection mode to `char` on every new click, so a later drag somewhere else starts a plain selection instead of reusing a stale `word` or `line` mode.
- **`MouseDrag1Pane`.** The drag still starts with `begin-selection`, because autoscroll on the top and bottom rows only works for a drag that tmux tracks itself. The inner `if` runs the repair only when `begin-selection` really has reset the selection mode.
- **`User900` and `User901`.** They move the cursor to the other end of the selection, which is still inside the original word or line, and select that word or line again. A `send-keys -X` command in a mouse binding first moves the cursor to the mouse position, which would select the wrong word. Sending a key that is bound in the copy-mode table runs the same commands without that move. No keyboard produces these two keys; any unused `User` key works.
- **`rectangle-off`.** Rectangle selection is off anyway, so the command is used only for what else it does in a mouse binding: it moves the cursor to the mouse position and recomputes the selection up to there, by word or by line.

**Autoscroll on tmux 3.8.** With `mode-keys vi`, tmux 3.8 scrolls a single line when a drag is held on the top or bottom row, and then stops. This is a tmux bug that affects every drag, including a plain one with the default bindings. The fix ([tmux/tmux#5667](https://github.com/tmux/tmux/pull/5667)) came after the 3.8 release, so it takes a newer tmux, not a different binding.

I checked these bindings on tmux 3.8 only, by sending mouse events to a test server. With `mode-keys vi`: double click then drag in both directions, a drag that starts right after the second click, triple click then drag up and down, drags that end on the top and bottom rows, a plain drag, and clicks without a drag all select and copy the expected text. Autoscroll I could only check with the same bindings in the `copy-mode` table under `mode-keys emacs`, where 3.8 does not have the bug: plain, word and line drags keep scrolling while the pointer rests on the top or bottom row, and stop when the button is released.
