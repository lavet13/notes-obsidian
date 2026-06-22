---
id: journal
aliases:
  - journal
tags: []
---

# Journal

## 2026-06-22 — WSL dev-env: tmux auto-start, clipboard mechanism, dotfile symlinks

**tmux auto-start ordering.** The auto-start `exec tmux` block must sit ABOVE p10k's
instant-prompt block in .zshrc — otherwise p10k grabs the TTY first and tmux dies with
"open terminal failed: not a terminal." Key insight: guard it with `[[ -t 1 ]]` (is fd 1
a tty?) so non-interactive/no-tty shells skip it cleanly. Deleted the stale duplicate
tmux block lower in .zshrc (it was below instant-prompt → would re-trigger the bug).

**Per-launch tmux opt-out.** Added a `NO_TMUX` escape hatch: `[[ -z "${NO_TMUX:-}" ]]`
in the guard, plus a second WezTerm launch_menu entry that runs
`wsl.exe -d Debian --cd ~ -- env NO_TMUX=1 zsh -li`. The `--` ends wsl.exe's own options;
the rest runs inside the distro. `env NO_TMUX=1` sets the var, interactive-login zsh
sources .zshrc, sees NO_TMUX, skips the auto-start → plain shell on demand.

**Clipboard — corrected understanding.** Copy reaches the Windows clipboard via OSC 52,
NOT via clip.exe. `set-clipboard external` (the default, confirmed via
`tmux show -gv set-clipboard`) makes tmux emit an OSC 52 escape sequence on every copy;
WezTerm decodes it onto the Windows clipboard. This fires for `copy-selection-and-cancel`
too — independent of `copy-command`. So `y` already works; removed the redundant
`if-shell … copy-command 'clip.exe'` line. Kept the vi `v`/`y` binds (they override
tmux's non-Vim defaults: Space/Enter, and v=rectangle-toggle).

**Dotfiles → symlinks.** Step 9 of wsl-setup.sh now symlinks (ln -sf) instead of copying,
guarded: `[[ -n "$CLONED_DOTFILES" ]]` → cp (temp-clone standalone run), else → ln -sf
(persistent repo). Live working copy: edits in ~ ARE edits in the repo, no drift.
Committed the executable bit on tmux-sessionizer at the source
(`git update-index --chmod=+x`, 100644 → 100755) and dropped the script's `chmod +x` —
a fresh clone now gets an executable file the symlink inherits.

**Open threads / resume:** none — verified working end to end (re-ran wsl-setup.sh, tmux
auto-starts, no-tmux launch entry works, clipboard yanks to Windows, sessionizer runs).

## 2026-06-17 — WSL dev-env: wsl-setup.sh hardening, tmux/zsh wiring, git workflow

### Covered (concept → key insight)
- **Sessionizer "opens & closes"**: `display-popup -E tmux-sessionizer` runs via a
  NON-interactive `zsh -c`, which sources `.zshenv` only — not `.zshrc`, where
  `~/.local/bin` was added. KEY: a pane's PATH ≠ the PATH a popup/`zsh -c` child gets.
  Fix: PATH export moved to `.zshenv`. Also: tmux auto-start belongs in interactive
  `.zshrc` so the server is born with the right PATH.
- **glob vs regex**: `find -name` = glob (`.` literal, `*` = any run); `fd` = regex
  (`.` = any char, `*` = zero+ of preceding). Same pattern, different meaning.
- **cp overwrite/merge**: `-n` deprecated on GNU (Debian warns) → use `--update=none`.
  `src/.` or `-rT` merge a dir's CONTENTS into dest without nesting.
- **trap**: `EXIT` is the catch-all — fires however the script leaves, so one
  `trap cleanup EXIT` covers every exit path; `ERR`/`INT`/`TERM` react to specific events.
- **mktemp -d**: race-safe, uniquely-named 0700 scratch dir vs mkdir's fixed name.
- **git auth**: HTTPS password auth was removed in 2021 → use SSH (`remote set-url
  git@github.com:…`). `id_ed25519` authenticates by default.
- **detached HEAD**: checking out a TAG detaches; for a live working copy, `git switch main`.
- **--ff-only**: pull only if it's a clean fast-forward; refuse (don't merge) on divergence.
- **lazy.nvim**: `:Lazy sync` UPDATES plugins + rewrites lazy-lock.json (the "churn");
  `:Lazy! restore` installs at the locked versions without rewriting → reproducible.

### Changed / built
- wsl-setup.sh: `trap cleanup EXIT` + `ERR` diagnostics; SSH repo URLs upfront +
  `ssh-keyscan github.com`; step 7 → live working copy (no tag checkout); step 8 →
  `git pull --ff-only`; graft via `cp -rT --update=none`; `.zshenv` copied + CRLF-stripped;
  pre-warm `sync` → `restore`; added shellcheck/info/bash-doc/cht.sh.
- `.zshenv` created (PATH); `.zshrc` tmux auto-start; `.tmux.conf` `prefix f` sessionizer;
  removed nvim `<C-f>` remap (single trigger now); default_prog = Git Bash, WSL via launch_menu.
- nvim-lsp: switched to `main`, fast-forwarded; ready to delete stale `nvim-0.11` branch.
- README regenerated WSL-only; decided to drop the Docker dev-container + Gemini entirely.
- Notes layout settled: per-topic workspace with `journal.md` (this) + `*-knowledge.md` (reference).
- Mentor prompt: generalized "Progress, handoffs & reference" with `wrap` + `ref` triggers.

### Open threads / next step
1. Commit + push repo cleanup: `git rm Dockerfile docker-compose.yml gemini-workflow.md`,
   `git rm -r .gemini`, drop README's old sections (done), commit.
2. Delete stale branch: `git push origin --delete nvim-0.11 && git branch -d nvim-0.11`.
3. `shellcheck wsl-setup.sh` and clear any findings.
4. Create `notes/dev-env/` in notes-obsidian + register it in obsidian.nvim; add
   `journal.md`, `bash-knowledge.md`, `git-knowledge.md`.
5. Apply the generalized section to mentor-main-prompt.md.

### Commit message
  chore: go WSL-only — drop Docker dev-container + Gemini

  - remove Dockerfile, docker-compose.yml, .gemini/, gemini-workflow.md
  - README rewritten for the wsl-setup.sh flow
  - wsl-setup.sh: SSH repo URLs + ssh-keyscan, live nvim working copy,
    notes pull --ff-only, Lazy restore (reproducible), trap cleanup/ERR
