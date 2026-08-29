---
id: python-knowledge
aliases: []
tags: []
---

# Python-Knowledge

## Poetry: in-project venv (and why the config seems ignored)

By default Poetry hides the venv in a global cache (`~/.cache/pypoetry/virtualenvs/<hash>`), which
editors/LSPs can't auto-find. Put it in the project as `./.venv`:

```bash
poetry config virtualenvs.in-project true   # global: create ./.venv per project
```

GOTCHA — this only applies when Poetry CREATES a venv. If a cache venv already exists Poetry reuses
it, and `poetry install` says "No dependencies to install or update" (nothing rebuilt), so `.venv`
never appears. Force a clean rebuild:

```bash
poetry env remove --all                                   # drop Poetry's known envs
rm -rf .venv "$(poetry config virtualenvs.path)/<name>"   # physically delete the cache venv it falls back to
poetry install --with dev                                 # now builds ./.venv (long install list)
poetry run python -c "import sys; print(sys.prefix)"      # verify → .../<project>/.venv
```

BIGGER GOTCHA — if `VIRTUAL_ENV` is set in the shell (an activated venv), Poetry USES it and ignores
`virtualenvs.in-project` entirely. Check `echo "$VIRTUAL_ENV"`, `unset VIRTUAL_ENV`, then rebuild.

## pyright: point the LSP at the venv

pyright resolves imports against an interpreter; on system Python it can't find your deps
(`reportMissingImports`). Point it at the project venv with a `pyrightconfig.json` at repo root:

```json
{ "venvPath": ".", "venv": ".venv" }
```

`venvPath` = dir CONTAINING the venv, `venv` = the venv folder name. `pyrightconfig.json` is also a
pyright ROOT MARKER, so it pins the workspace root too. Restart the LSP after (`:LspRestart`). Keep
it gitignored — local env layout, not project code. (Equivalently, set the nvim client
`settings.python.pythonPath = ".venv/bin/python"`, but that applies globally, not per-repo.)
