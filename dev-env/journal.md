---
id: journal
aliases:
  - journal
tags: []
---

# Journal

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
