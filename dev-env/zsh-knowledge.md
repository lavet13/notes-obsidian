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

## Word splitting — zsh splits command substitution, NOT parameter expansion (bash splits both)

IFS = Internal Field Separator (default: space, tab, newline). bash splits EVERY
unquoted expansion on IFS; zsh does not — it hands you the value(s) intact.

```zsh
demo() { for a in $*; do echo " - $a"; done }
demo apple "two words" peal
#   zsh  → apple | "two words" | peal   (3 words — the string stays whole)
#   bash → apple | two | words | peal   (4 words — split on IFS)
```

Consequences in zsh: swapping `$*` for `$@` changes nothing (neither splits), and a
filename with spaces survives unquoted. To split ON PURPOSE, ask with a param flag:

```zsh
str="two words"
for w in ${=str};      do echo " - $w"; done   # =        → IFS word-split (bash-like)
for w in ${(s: :)str}; do echo " - $w"; done   # (s:SEP:) → split on a given separator
for w in ${(s:,:)csv}; do echo " - $w"; done   # e.g. split CSV on commas
```

```zsh
# The asymmetry (SH_WORD_SPLIT off = the default):
v="a b c"
for w in $v;              do echo "[$w]"; done   # PARAMETER → not split → [a b c]
for w in $(echo "a b c"); do echo "[$w]"; done   # COMMAND sub → split → [a] [b] [c]
for w in "$(echo a b c)"; do echo "[$w]"; done   # quoted → back to one word → [a b c]
```

Gotcha: `setopt shwordsplit` forces bash-style splitting everywhere — avoid, it
surprises you in unrelated code. Prefer per-expansion `${=var}` / `${(s:X:)var}`.

## Globbing — NOMATCH error and the `(N)` qualifier

By default zsh has NOMATCH on: a glob that matches nothing raises `no matches found`
and ABORTS the command — it never runs (bash instead passes the literal pattern
through). The error is emitted during EXPANSION, before the command and its
redirections are set up, so `&>/dev/null` cannot suppress it.

```zsh
ls *.mp3               # no matches → "zsh: no matches found: *.mp3"  (ls never runs)
ls *.mp3 &>/dev/null   # STILL errors — the abort happens before the redirection applies

# Fixes:
for f in *.mp3(N); do ...; done   # (N) = NULL_GLOB for THIS glob → vanishes if empty (idiomatic)
setopt null_glob                  # global: unmatched globs expand to nothing, everywhere
setopt nonomatch                  # global: pass the literal like bash (rarely what you want)
```

`(N)` is one of many glob qualifiers — the trailing `(...)` that filters or modifies a
match: `(.)` plain files, `(/)` dirs, `(@)` symlinks, `(om[1])` newest. `(N)` just adds
"empty is fine." So `clipfiles ~music/*.mp3(N)` degrades quietly instead of erroring.
