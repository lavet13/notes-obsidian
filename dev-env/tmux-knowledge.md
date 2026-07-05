---
id: tmux-knowledge
aliases: []
tags: []
---

## Clipboard: how a copy reaches the system clipboard

There are TWO independent routes tmux -> system clipboard. Don't conflate them.

1. OSC 52 + set-clipboard (the DEFAULT path)
   When you copy in copy mode, tmux base64-encodes the selection and emits an
   OSC 52 escape sequence (ESC ] 52 ; c ; <base64> BEL) to the OUTER terminal,
   which drops it on the system clipboard. Fires on EVERY copy, including
   copy-selection-and-cancel. No helper binary; works over SSH; one-directional
   (sets the clipboard, can't read it). Needs the terminal's Ms capability
   (tmux adds it for TERM=xterm*) and OSC 52 support (WezTerm has both).

2. copy-command / copy-pipe (the external-tool path)
   Only copy-pipe* bindings use it. `set -s copy-command 'clip.exe'` then
   copy-pipe-and-cancel (no arg) pipes the selection to that program.

```bash
tmux show -gv set-clipboard      # external (default) or on -> OSC 52 is active
```

Gotcha: copy-selection-and-cancel does NOT use copy-command -- but it STILL reaches
the clipboard via OSC 52. So on a WezTerm/xterm-class terminal, a plain
`y = copy-selection-and-cancel` already yanks to the system clipboard; a separate
clip.exe/copy-command line is redundant (a harmless fallback for no-OSC-52 terminals).

### set-clipboard values

- external (default since 2.6): tmux sets the OUTER terminal's clipboard on copy;
  does NOT accept OSC 52 from apps running inside tmux.
- on: same as external PLUS accepts OSC 52 from inside apps (needed for nested tmux).
- off: no OSC 52 at all.

---

## Auto-start tmux from .zshrc (ordering matters)

```bash
# MUST sit ABOVE p10k's instant-prompt block. p10k grabs the TTY early; if tmux
# starts after it, tmux dies with "open terminal failed: not a terminal" and p10k
# warns about console output during init.
if [[ -z "$TMUX" ]] && [[ -z "${NO_TMUX:-}" ]] && [[ -o interactive ]] && [[ -t 1 ]] && command -v tmux &>/dev/null; then
  exec tmux new-session -A -s main
fi
```

- `[[ -z "$TMUX" ]]`      not already inside tmux (the inner shell re-sources .zshrc).
- `[[ -z "${NO_TMUX:-}" ]]` escape hatch: a launch entry can set NO_TMUX=1 to opt out.
- `[[ -t 1 ]]`            fd 1 is a real tty -- the direct cure for "not a terminal".
- `exec`                  replace this zsh with tmux (exit tmux = close terminal).

Symptom -> cause: "open terminal failed: not a terminal" = tmux started with no real
TTY (below the instant-prompt block, or in a no-pty shell).

---

## copy-mode-vi bindings

```bash
set-window-option -g mode-keys vi   # vi NAVIGATION in copy mode (hjkl, /, w/b)
# Default vi copy keys are Space (begin-selection) and Enter (copy-and-cancel);
# on recent tmux, v defaults to rectangle-toggle, NOT begin-selection.
bind-key -T copy-mode-vi 'v' send -X begin-selection        # remap to match nvim
bind-key -T copy-mode-vi 'y' send -X copy-selection-and-cancel
```

Gotcha: mode-keys vi only sets navigation. If you want v/y like nvim visual mode you
must bind them; otherwise v toggles rectangle selection and you copy with Enter.
