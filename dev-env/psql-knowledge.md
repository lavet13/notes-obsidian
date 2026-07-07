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
docker compose exec postgres psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"
# -U = user, -d = database. The prompt becomes  dbname=#
```

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
