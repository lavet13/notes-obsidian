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

## Dropping a table: when it cascades into data you meant to keep

A DROP cascades into a table you're KEEPING only when that kept table has an
`onDelete: Cascade` foreign key pointing AT the table you're dropping. FKs
pointing OUTWARD from the dropped table are harmless — they just vanish with it.

Prisma drops FK constraints first (child before parent), which is why a clean
teardown looks like:

```sql
ALTER TABLE "child" DROP CONSTRAINT "child_parent_id_fkey";
DROP TABLE "child";
DROP TABLE "parent";
```

Safe design for log / snapshot tables: store a plain denormalized value
(e.g. `managerChatId BigInt`, NO `@relation`) instead of a real FK to the entity.
Then dropping the entity can't touch the log — history survives. This is why
NotificationLog (plain managerChatId column) was unaffected when Manager dropped.

Verify before dropping: a migration that DROPS a non-empty table warns
("about to drop X, which is not empty (N rows)"). That warning is expected when
you've already backed up (pg_dump -t) — it's not an error, it's the safety notice.
