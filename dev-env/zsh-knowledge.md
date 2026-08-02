---
id: zsh-knowledge
aliases:
  - zsh-knowledge
tags:
  - zsh
  - reference
---

## Named directories (`hash -d`) — zsh only

Give a directory a short name that works everywhere a path does — not just a
`cd` shortcut like an alias.

```zsh
hash -d notes=~/notes            # define: name -> path
hash -d nvc=~/.config/nvim
hash -d work=~/workspace
```

Now `~notes` expands to the path in ANY command, and the prompt shows the short
form:

```zsh
cd ~notes                # jump there
nvim ~work/todo.md       # use mid-path, not just for cd
cp file ~nvc/            # anywhere a path is expected
ls ~notes                # tab-completion works: ~no<Tab> -> ~notes
```

Why it beats `alias notes='cd ~/notes'`:

- An alias that's just a path (`alias notes='~/notes'`) does NOT cd — the shell
  tries to EXECUTE the directory. `hash -d` is the correct tool.
- Works as a path fragment (`~work/sub/file`), which a cd-alias can't.
- p10k / prompt path display collapses the real path to `~notes`, keeping the
  prompt short.
- Tab-completes after the `~name`.

Gotcha: it's `~name` (tilde-prefixed), not bare `name`. `cd notes` still fails;
`cd ~notes` is the form. Put the `hash -d` lines in .zshrc (ordering vs the p10k
instant-prompt block doesn't matter — they don't write to the terminal).

## Checking .zshrc for errors without running it

    zsh -n ~/.zshrc      # -n = no-exec: PARSE only, report syntax errors, run nothing
    zsh -il -c exit      # load a real interactive login shell → surfaces runtime errors too

`zsh -n` catches STATIC / syntax errors only (unbalanced quotes, a missing `fi`) — found
by parsing alone. It does NOT catch LOGIC errors: a valid line with the wrong meaning.
e.g. `hash -d video=~/yt/video` (see Named directories) is syntactically perfect —
`hash -d` never checks the path exists — so a typo (video vs videos) passes -n clean and
only fails at USE time: `cd ~video` → no such file or directory.

## History expansion (`!`) fires before parameter expansion — interactive only

Interactive zsh/bash read `!` as the history trigger (`!!`, `!vim`, `!42`) on the RAW line,
BEFORE variables expand. So a `!` inside a word — `/proc/$!/cmdline` — is taken as a history
event, not text:
zsh: event not found: /cmdline
Scripts and `sh -c` have history expansion OFF, so it only bites INTERACTIVELY — a snippet
that ran fine in a script can break pasted live.
Fix — capture the special var (`$!` = last bg PID, see bash-knowledge Special Variables) into
a plain var immediately, then use that (no `!` downstream; also the next bg job overwrites `$!`):

```zsh
sh -c 'sleep 300; true' &
pid=$!                            # grab NOW; lines below have no '!' to trip on
tr '\0' ' ' < /proc/$pid/cmdline
```

Session off-switch: `setopt nobanghist` (zsh) / `set +H` (bash). Capturing is still better.

## Getting help for builtins — `man` won't; use `run-help` / `man zshbuiltins`

A builtin (`umask`, `cd`, `export`, `read`, `setopt`, `hash`) lives inside the shell, so it
has no standalone man page — `man umask` shows a generic POSIX stub, not zsh's real behavior.
Ask the shell instead:

```zsh
run-help umask       # zsh; may need: autoload -Uz run-help   (OMZ binds Esc-h on the line)
man zshbuiltins      # every zsh builtin in one page (also: man zshall)
help umask           # bash's equivalent
type umask           # → "umask is a shell builtin"  (tells you it's a builtin at all)
```

Learning ladder: tldr (examples) → `<cmd> --help` (YOUR version's flags) → man/info (full
semantics) → run-help / man zshbuiltins the moment it's a builtin.

## zsh startup files — load order & which runs when

"rc" = "run commands" (a CTSS _runcom_ fossil): the startup file(s) a shell RUNS on launch.
zsh reads a SET, in this fixed order; the system file runs just before each per-user `~/.*`.
Source of truth: `man zsh`, "STARTUP/SHUTDOWN FILES".

```text
FILE          runs for…                                   system (Arch: /etc/zsh/)
~/.zshenv     EVERY zsh: login, interactive, AND scripts  /etc/zsh/zshenv
~/.zprofile   LOGIN shells only, before .zshrc            /etc/zsh/zprofile
~/.zshrc      INTERACTIVE shells — the main one           /etc/zsh/zshrc
~/.zlogin     LOGIN shells only, after .zshrc             /etc/zsh/zlogin
~/.zlogout    LOGIN shells, on logout                     /etc/zsh/zlogout
```

- `.zshrc` = what "my rc" means 95% of the time: aliases, prompt, keybinds, options, completion
  — anything you experience while typing interactively.
- By shell type: login → zshenv, zprofile, zshrc, zlogin. Interactive non-login → zshenv, zshrc.
  Script (`zsh foo.sh`) → zshenv ONLY.
- GOTCHA: `.zshenv` runs for non-interactive scripts too → keep it TINY (essential env vars
  only); heavy work there slows every script and can break tools that spawn zsh.
- umask home: a per-user default in `.zshrc`/`.zprofile` covers your shells; system-wide
  defaults come from PAM (`pam_umask`) or `/etc/profile`, not your rc — where a umask you
  didn't set is hiding.
