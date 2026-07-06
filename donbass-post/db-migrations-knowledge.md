---
id: db-migrations-knowledge
aliases: []
tags: []
---

## Soft-delete convention: revokedAt / nullable timestamp

`revoked_at DateTime?` — `null` = ACTIVE, a timestamp = revoked-at-that-moment.
The field NAME sounds like "true = revoked"; the convention is the opposite, so
always say "null means active" before writing the comparison. Pair with
`granted_at` for a full [granted, revoked) lifespan (audit/history) — a boolean
flag can't answer "who held this role last March."

## Staged data migration (zero-downtime table swap)

Order: backfill → migrate writes → migrate reads → drop old. NEVER swap reads
first (new table empty = silent breakage). Backfill gated bootstrap-only (run
only when new table empty) so it can't resurrect rows deleted at runtime from
the now-frozen old table. Reads and their companion read-modify-write commands
must move together, or you get split-brain drift.

## Undoing a migration (Prisma) — there is no clean "down"

Prisma migrations are FORWARD-ONLY — there's no auto-generated down migration.
To undo a schema change you write a NEW migration that reverses it (same as
`git revert`: you move forward to go back, you don't rewrite history).

```bash
# The wrong instinct (destructive, dev-only): reset the whole DB to migrations
prisma migrate reset          # DROPS the db, re-applies all migrations + seed.
                              # NEVER in production — it deletes all data.

# The right way to undo in prod: create a new migration that reverses the change
# e.g. you dropped a column and regret it -> a new migration re-adds it.
prisma migrate dev --name revert_drop_manager_tables
```

Key point: a migration that DROPS a table is irreversible for the DATA — the
rows are gone even if you re-add the table. So before an irreversible migration
(dropping a table), the "undo" is a BACKUP, not a migration. Back up, then drop.
