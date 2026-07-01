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

---

# Anki's cards

Card 1 - soft-delete convention (recall)

Front: Soft-delete - with a revokedAt DateTime? column, which value means ACTIVE?
Back: null means ACTIVE. A timestamp means revoked-at-that-moment. The field NAME sounds like "true = revoked", so the convention is the opposite of what it reads like - always say "null means active" before writing the comparison.

Card 2 - soft-delete comparison (application)

Front: Soft-delete - you are writing a guard that rejects a revoked role. Is the condition revokedAt === null or revokedAt !== null?
Back: revokedAt !== null rejects (a timestamp = revoked). === null would reject ACTIVE roles - the inverted-check bug. Say "null means active" first, every time.

Card 3 - grantedAt pairing (recall)

Front: Soft-delete - why pair revokedAt with a grantedAt timestamp instead of using a boolean flag?
Back: The pair gives a full [granted, revoked) lifespan for audit/history. A boolean flag cannot answer "who held this role last March" - it only holds the current state.

Card 4 - staged migration order (recall)

Front: Zero-downtime table swap - what is the correct order of the four stages?
Back: backfill -> migrate writes -> migrate reads -> drop old. Never swap reads first (the new table is empty until backfilled = silent breakage).

Card 5 - reads-first hazard (application)

Front: Data migration - why must you never migrate the READS to a new table before backfilling and migrating writes?
Back: The new table is empty until backfilled, so reading it first returns nothing - silent breakage (e.g. zero notifications sent) with no error. Reads must come after the data and writes are in place.

Card 6 - gated backfill (recall)

Front: Data migration - why gate a backfill to run only when the new table is empty (bootstrap-only)?
Back: So it cannot resurrect rows that were deleted at runtime. If the backfill re-ran on every deploy, it would re-import from the now-frozen old table, undoing runtime deletions. Empty-table gate = runtime stays authoritative.

Card 7 - read-modify-write coupling (application)

Front: Data migration - an append command reads current subscriptions from the OLD table and writes the new set to the NEW table. What breaks?
Back: Split-brain drift / data loss. The old table is frozen, the new one is live, so current is stale - appending C to a stale [A] silently drops B. Reads and their companion read-modify-write commands must migrate TOGETHER, not separately.
