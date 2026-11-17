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
# and start over on a new click
bind-key -T copy-mode-vi MouseDragEnd1Pane send-keys -X copy-selection-no-clear
bind-key -T copy-mode-vi MouseDown1Pane select-pane \; send-keys -X clear-selection

# double click selects a word, triple click selects a line, both stay in copy mode
bind-key -T root DoubleClick1Pane {
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
bind-key -T copy-mode-vi DoubleClick1Pane select-pane \; send-keys -X select-word \; send-keys -X copy-pipe-no-clear
bind-key -T copy-mode-vi TripleClick1Pane select-pane \; send-keys -X select-line \; send-keys -X copy-pipe-no-clear

# tmux 3.8 workaround: re-apply the word or line selection mode
# that begin-selection has just reset to char
bind-key -T copy-mode-vi MouseDrag1Pane {
  select-pane
  if -F '#{==:#{selection_mode},word}' {
    send-keys -X begin-selection
    send-keys -X selection-mode word
  } {
    if -F '#{==:#{selection_mode},line}' {
      send-keys -X begin-selection
      send-keys -X selection-mode line
    } {
      send-keys -X begin-selection
    }
  }
}
```

The `MouseDown1Pane` binding is required by the workaround. `clear-selection` resets the selection mode to `char` on every new click, so a later drag somewhere else starts a plain selection instead of reusing a stale `word` or `line` mode.

I checked these bindings on tmux 3.8 only, by sending mouse events to a test server: double click then drag in both directions, triple click then drag up and down, a plain drag, and clicks without a drag all select and copy the expected text.
