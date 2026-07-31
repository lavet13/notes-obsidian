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
