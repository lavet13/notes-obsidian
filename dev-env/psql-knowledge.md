---
id: psql-knowledge
aliases:
  - psql-knowledge
tags: []
---

# psql Knowledge

Raw Postgres CLI reference — cross-project (any DB-backed work, not Prisma-specific).

## Connecting inside a Docker Compose stack

```bash
# Let the CONTAINER expand its own env vars — single quotes + sh -c defers expansion.
docker exec -it donbass-post-db sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"'

# One-shot query, no REPL:
docker exec -it donbass-post-db sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT name FROM roles;"'

# `docker compose exec postgres …` also works — same quoting rule applies.
# -it = interactive TTY, needed for the psql prompt (omit for one-shot in scripts/CI).
```

```bash
docker compose exec postgres psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"
#                                    ^^^^^^^^^^^^^^^^ double-quoted → YOUR shell expands this,
#                                    before docker ever runs. POSTGRES_USER lives in
#                                    apps/telegram-bot/.env, loaded into the CONTAINER via
#                                    env_file — it's not in your shell. → expands to empty →
#                                    `psql -U -d donbass_post` → psql reads "-d" as the username.
```

> **GOTCHA — where does `$VAR` expand?**
> `docker exec db psql -U "$POSTGRES_USER"` → **your shell** expands it BEFORE docker runs.
> The var lives in the container (via `env_file`), not your shell → expands to EMPTY →
> `psql -U -d mydb` → psql reads `-d` as the username. Confusing error, wrong layer.
> `sh -c '…$VAR…'` (SINGLE quotes) passes the string through literally; the container's
> shell expands it against the container's env. Same rule for `docker run`, `ssh host '…'`,
> and any `bash -c` — **single quotes = expand THERE, double quotes = expand HERE.**

## Meta-commands (psql's own backslash commands — NO semicolon)

- `\l` — list all databases
- `\c dbname` — connect to / switch database
- `\dt` — list tables in the current schema
- `\dt *.*` — list tables in ALL schemas
- `\d tablename` — describe a table: columns, types, nullability, indexes, FKs
- `\d+ tablename` — same, with more detail (storage, comments)
- `\di` — list indexes
- `\dn` — list schemas
- `\du` — list roles / users
- `\dv` — list views
- `\df` — list functions
- `\x` — toggle expanded display (rows as key/value blocks — great for wide tables)
- `\q` — quit
- `\?` — help for backslash commands
- `\h SELECT` — SQL syntax help for a statement

## The one you forget: reading a table's structure

`\d tablename` is the workhorse — columns, types, defaults, indexes, and foreign-key
constraints. `\d+` adds storage/comments. Quick row peek: `SELECT * FROM t LIMIT 5;`
(SQL — needs the semicolon).

## Gotchas

- Backslash commands (`\dt`) take NO semicolon; SQL statements (`SELECT ...`) NEED one.
- `\dt` shows only the CURRENT schema (usually `public`). Use `\dt *.*` for all.
- Names case-fold to lowercase unless double-quoted: `\d Users` looks for `users`.
  Quote to force case: `\d "Users"`.
- Inside a container you're already the DB user via `-U`; no password prompt if
  local trust/peer auth is set.
