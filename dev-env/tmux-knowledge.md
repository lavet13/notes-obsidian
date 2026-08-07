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

## `SWAYSOCK` staleness in a long-lived server

The tmux SERVER captures the environment once, at first start, and hands that frozen copy to every
pane. sway's IPC socket path embeds sway's PID (`sway-ipc.1000.<PID>.sock`), so relaunching sway
mints a new socket while tmux keeps pointing at the dead one — `swaymsg` inside tmux then fails with
`Unable to connect to …sway-ipc…sock`, though it works in a fresh (non-tmux) shell.

```tmux
# Durable fix: refresh these from the attaching client on attach. -g global, -a APPEND (keep tmux's
# defaults, which already carry DISPLAY/SSH_*). Then DETACH + REATTACH (only new panes get refreshed).
set -ga update-environment "SWAYSOCK"
set -ga update-environment "WAYLAND_DISPLAY"
```
Immediate fix without the above: `tmux kill-server` and restart inside the live sway session.

## Pane nav: `-r` is a repeat-window trap

`bind -r` marks a binding REPEATABLE — after it fires, tmux stays in a `repeat-time` window (default
500 ms) where a BARE `h/j/k/l` re-triggers *without* the prefix. So `prefix k` then `h` reads the `h`
as "repeat → go left", bouncing you back. For one move per prefix, use plain `bind`:

```tmux
bind h select-pane -L   ;  bind j select-pane -D
bind k select-pane -U   ;  bind l select-pane -R
```
(No-prefix `bind -n M-h …` was dead separately: wezterm sends Left-Alt as a *compose* key, not Meta,
unless `send_composed_key_when_left_alt_is_pressed = false`.)

<!-- Docs: https://man.archlinux.org/man/tmux.1 -->

## Open a URL from a pane (no mouse)

`prefix + u` → fzf-pick any URL in the pane (including scrollback) and open it via xdg-open.

```tmux
bind u display-popup -E "tmux capture-pane -J -p -S -3000 | grep -oE 'https?://[^ ]+' | sort -u | fzf --reverse | xargs -r xdg-open"
```

- `display-popup -E "…"` — open a floating popup running the command; **`-E`** closes the popup
  automatically as soon as the command exits (without it the popup lingers showing a dead shell).
- `capture-pane` — dump the pane's text so we can grep it:
  - **`-J`** join wrapped lines — a long URL soft-wrapped across two rows is captured as ONE line,
    so grep sees the whole URL instead of two broken halves.
  - **`-p`** print the capture to stdout (the pipe) instead of into a tmux paste-buffer.
  - **`-S -3000`** set the start line 3000 rows back into the scrollback, so it searches history,
    not just the visible screen. (`-S -` would mean "the very top"; a number caps the cost.)
- `grep -oE 'https?://[^ ]+'` — pull the URLs out:
  - **`-o`** print ONLY the matched URL, not the whole line it sat on.
  - **`-E`** extended regex, so `?`, `+`, `|` work unescaped (`https?` = the "s" is optional).
- `sort -u` — **`-u`** unique: collapse duplicate URLs so the picker lists each once.
- `fzf --reverse` — **`--reverse`** draws the prompt at the top and results below (top-down),
  which reads naturally in a small popup.
- `xargs -r xdg-open` — feed the picked URL to xdg-open; **`-r`** ("no-run-if-empty") means if you
  Esc out of fzf and nothing is selected, xdg-open is NOT run with an empty arg (avoids an error).

Also: **Shift+click** a link bypasses tmux's `mouse on` capture → wezterm's own URL handler opens it.

<!-- Docs: https://man.archlinux.org/man/tmux.1 -->
