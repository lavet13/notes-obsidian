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

### Switch a clone's remote HTTPS -> SSH (after the fact)

```bash
git remote -v                                          # see current urls
git remote set-url origin git@github.com:OWNER/REPO.git
git remote -v                                          # verify it took
```

Common when a clone happened over HTTPS (e.g. before an SSH key was in place).
Gotcha: easy to drop the `OWNER/` segment — `git@github.com:repo.git` (no owner)
is wrong and only fails later on push. Always re-check with `git remote -v`.

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

## Undoing changes — code (working tree, staged, committed)

The mental model: three places a change can live — working tree, staging area,
committed history — and a different tool for each.

```bash
# WORKING TREE (edited, not staged): throw away uncommitted edits to a file
git restore <file>              # discard working-tree changes (was: checkout -- <file>)
git restore .                   # discard ALL uncommitted working-tree changes

# STAGED (git add'd, not committed): unstage but KEEP the edits
git restore --staged <file>     # move it out of the index, edits remain in working tree

# COMMITTED — two very different tools:
# 1. revert = make a NEW commit that undoes an old one. History preserved.
#    SAFE on pushed/shared branches (doesn't rewrite history).
git revert <commit>             # opens editor for the "Revert ..." message
git revert HEAD                 # undo the most recent commit as a new commit

# 2. reset = MOVE the branch pointer back. Rewrites history.
#    ONLY on local/unpushed commits (force-push needed otherwise).
git reset --soft HEAD~1         # undo last commit, KEEP changes staged
git reset --mixed HEAD~1        # undo last commit, keep changes UNSTAGED (default)
git reset --hard HEAD~1         # undo last commit AND discard the changes (destructive)
```

Rule of thumb: **pushed → `revert`** (new commit, safe), **local-only → `reset`**
(moves the pointer). `--hard` is the only one that destroys work — reach for it
last, and never on anything you've pushed.

## Amend vs revert vs reset (which "undo" do I want?)

- Fix the LAST commit's message/content, not pushed → `git commit --amend`
  (Fugitive: `cw` from `:Git` status).
- Undo a commit that's already PUSHED → `git revert <commit>` (new inverse commit).
- Undo a LOCAL commit, keep the work → `git reset --soft/--mixed HEAD~1`.
- Nuke a local commit and its changes → `git reset --hard HEAD~1`.

## Fork workflow: `origin` vs `upstream`

Convention: `origin` = YOUR fork (push here), `upstream` = the source repo (pull from here).
A direct clone of someone else's repo has `origin` on THEIR repo — fix it before contributing:

```bash
git remote rename origin upstream                 # their repo → 'upstream'
git remote add origin git@github.com:ME/REPO.git  # my fork → 'origin'
git remote -v                                     # verify fetch/push urls per remote
```

Sync my fork's main with upstream later (run FROM main — pull acts on the CURRENT branch):

```bash
git switch main
git fetch upstream                 # download upstream's commits, no merge yet
git pull --ff-only upstream main   # fast-forward only (see `pull --ff-only`); refuse on divergence
git push origin main               # push synced main to my fork
```

Forks start with NO Actions secrets and workflows disabled — deliberate, so a fork can't push to
the source's registry.

## force-with-lease vs force (safe history rewrite)

Rewriting a commit (amend/rebase) makes a NEW hash; the remote still has the old one, so a plain
`git push` is rejected (non-fast-forward). Force is required — use the lease variant:

```bash
git push --force-with-lease   # overwrite ONLY if remote still points where I last fetched
git push --force              # overwrite unconditionally — can clobber someone else's push
```

`--force-with-lease` aborts if the remote moved since my last fetch, so it can't silently destroy
work. Timing rule once a PR exists:
- BEFORE review / no PR yet → amend + `--force-with-lease` (keep one tidy commit).
- AFTER review starts → ADD a new commit + plain `git push`; rewriting breaks an in-progress review.
Only ever rewrite a branch that's exclusively mine.

## Splitting one commit into several

Dissolve the commit but keep every change, then re-commit in groups:

```bash
git reset --soft HEAD~1     # undo commit; changes stay STAGED, nothing lost
git restore --staged .      # unstage → changes now in the working tree
git add -p <file>           # stage only the hunks for change #1 (see `git add -p`)
git commit -m "fix: ..."    # commit #1
git add -A                  # everything remaining
git commit -m "feat: ..."   # commit #2
```

Verify: `git log --oneline -2`, then `git show --stat HEAD~1`. Reverse if pushed: history rewrite
→ `git push --force-with-lease`.

## Staging selectively: `git add -p`

Walks each changed HUNK and asks whether to stage it. Prompt keys:
- `y` stage hunk / `n` skip
- `s` SPLIT into smaller hunks (only if an unchanged line separates the changes)
- `e` EDIT by hand — delete the `+`/`-` lines you don't want, keep the rest
- `q` quit
Fugitive equivalent: in `:Git` status, `=` expands a file's diff, then `s` on a hunk (or
visual-select lines + `s`).

## Cherry-pick a commit onto another branch

Copies a commit (by hash) onto wherever HEAD is — good for lifting one commit onto a fresh branch:

```bash
git switch -c fix/login-type main   # new branch off main
git cherry-pick 581849d             # replay just that commit here
```

Stacking: a commit that depends on an earlier one branches off THAT branch, not bare main — else
it conflicts with / duplicates the missing prerequisite.

## `rebase --onto`: move only the commits AFTER a cutoff

Plain `git rebase <newbase>` replays every commit since the MERGE BASE (common ancestor) — in a
stacked branch that wrongly includes a commit already merged upstream.

`git rebase --onto <newbase> <cutoff> [branch]` sets the cutoff manually:
- `<newbase>` = where commits land
- `<cutoff>`  = commits up to AND INCLUDING this are EXCLUDED

```bash
# feat branched off fix; fix already merged into upstream main.
# Replay ONLY feat's own commits onto the new main, dropping the redundant fix:
git rebase --onto main fix/login-type feat/token-refresh
```

Without `--onto`, plain rebase replays `fix` too and collides with the merged copy.

## Renaming a branch (local + remote)

A branch name is local; the remote isn't "renamed" — push the new name, delete the old:

```bash
git branch -m new-name              # rename current (-M forces over an existing name)
git push origin --delete old-name   # remove the stale branch from the fork
git push -u origin new-name         # push new name + set tracking (first push of this name)
```

Gotcha: renaming a branch that has an OPEN PR CLOSES that PR (GitHub ties the PR to the branch
name). Rename BEFORE opening the PR, or expect to reopen.
