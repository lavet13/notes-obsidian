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

## Prefix key: capture & `send-prefix`

The prefix is the key tmux intercepts to begin a command. Once set, tmux ALWAYS swallows
a single press and waits for the next key — so the prefix key can NEVER fall through to
the program in the pane as its own binding.

```bash
unbind C-b                       # drop the default prefix binding
set-option -g prefix C-Space     # new prefix (default was C-b)
bind-key C-Space send-prefix     # OPTIONAL: double-tap sends ONE literal C-Space to the app
```

- `send-prefix` is only needed when the prefix shadows a key you use IN the app. Classic
  case: prefix `C-a` shadows readline's beginning-of-line — bind `C-a send-prefix` and
  double-tap to reach line-start. Pick a prefix you never need in-app (C-Space) and you
  can drop send-prefix entirely.
- GOTCHA: a prefix the SYSTEM grabs never reaches tmux. `Ctrl+Space` is a common IME /
  keyboard-layout toggle (especially with a second Cyrillic layout) — the compositor eats
  it first. Test outside tmux; if the layout flips, rebind that toggle or pick another prefix.

---

## Subcommands & flags reference

```bash
# new-session (alias: new) — create a session
tmux new-session -s NAME        # -s = session name
tmux new-session -A -s NAME     # -A = attach if NAME exists, else create (idempotent)
tmux new-session -d -s NAME     # -d = detached: create but DON'T attach this client
tmux new-session -c DIR -s NAME # -c = starting directory for the first window
#   flags stack:  -As NAME (attach-or-create)   -ds NAME (create detached)

# switch-client — move the CURRENT client between sessions
tmux switch-client -t NAME      # -t = target; the in-tmux "attach" — no nesting

# has-session — existence guard (sets exit code, prints nothing)
tmux has-session -t NAME 2>/dev/null   # exit 0 if NAME exists, non-zero otherwise

# kill
tmux kill-session -t NAME       # one session
tmux kill-server                # ALL sessions/panes (no reverse — just relaunch tmux)

# config / keys
tmux source-file ~/.tmux.conf   # re-read a config file (the `prefix r` idiom)
tmux send-prefix                # send one literal prefix keystroke to the pane
```

Option/bind flags used in this config:

```bash
set-option -g  ...   # -g global (all sessions); -s server-level; -w window;
                     # -a append to a value;  -u unset back to default
bind-key -T TABLE k  # -T = key-table (e.g. copy-mode-vi); -n = root table (NO prefix needed)
unbind k             # remove a binding
display-popup -E "cmd"   # floating popup; -E closes it when cmd exits (sessionizer uses this)
```

Targeting & the dot rule:

```bash
# Target syntax is  session:window.pane  (e.g.  main:1.2 ).
# → that's WHY a session name can't contain '.' — it's the window.pane separator.
name="${name//./-}"      # sanitize a dir name into a legal session name (or: tr . -)
```

---

## `ts` — named create-or-switch session helper

Lives in `.zshrc`; complements `t` (always "main") and the `prefix f` sessionizer.
Expansion mechanics (`${1:-…}`, `${PWD##*/}`, `${x//./-}`) are in [[bash-knowledge]].

```bash
ts() {
  local name="${1:-${PWD##*/}}"   # arg 1, else current dir's basename
  name="${name//./-}"             # '.' is illegal in session names → '-'
  if [[ -n "$TMUX" ]]; then       # already inside tmux?
    tmux new-session -ds "$name" 2>/dev/null  # create detached if missing (ignore "exists")
    tmux switch-client -t "$name"             # hop over — NO nested-session attach
  else
    tmux new-session -As "$name"              # outside: attach-or-create
  fi
}
```

Key decision: the `$TMUX` branch. Outside tmux, `-A` just attaches. INSIDE tmux, attaching
would nest a session in a session (tmux warns), so create detached + `switch-client` instead.
