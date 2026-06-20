---
id: git-knowledge
aliases:
  - git-knowledge
tags: []
---

# Git-Knowledge

## Authentication: HTTPS vs SSH

GitHub removed password auth for HTTPS Git in 2021 — the password prompt can never succeed.
Use SSH:

```bash
git remote set-url origin git@github.com:USER/REPO.git   # switch an existing clone
ssh -T git@github.com                                    # test → "Hi USER! ..."
```
`id_ed25519` is offered by default. The key must be registered on your GitHub account, and
`github.com` must be in `~/.ssh/known_hosts` (seed non-interactively with `ssh-keyscan`).

---

## Detached HEAD vs branch

Checking out a TAG or commit detaches HEAD — commits made there are unreachable later.
For a working copy you edit and push, be on a branch:

```bash
git switch main          # leave detached HEAD, move to the branch (clearer than checkout)
git switch -c feature    # create + switch
```

---

## pull --ff-only

A `pull` = `fetch` + `merge`. Fast-forward = local is strictly behind remote, so git just
slides the pointer forward (no merge commit).

```bash
git pull --ff-only       # only if it's a clean fast-forward; refuse on divergence
```
Safe for scripts/notes: it stops loudly instead of silently creating a merge.

---

## Deleting a merged branch

```bash
git log --oneline main..nvim-0.11    # commits in nvim-0.11 NOT in main (empty = safe to drop)
git push origin --delete nvim-0.11   # delete remote branch
git branch -d nvim-0.11              # -d = safe: refuses if NOT merged (-D forces)
```

---

## Cross-boundary / multi-machine git config

```bash
git config --global --add safe.directory '*'   # stop "dubious ownership" on Windows↔WSL repos
git config --global core.autocrlf input         # CRLF→LF on commit, never inject on checkout
# .gitattributes `* text=auto eol=lf` OVERRIDES core.autocrlf — the authoritative per-repo rule
```

---

## Run git as if from another dir

```bash
git -C /path/to/repo status    # -C = "cd there first" — no need to cd yourself
```
