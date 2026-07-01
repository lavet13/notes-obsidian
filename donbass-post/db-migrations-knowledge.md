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
