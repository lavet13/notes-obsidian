---
id: knowledge
aliases:
  - knowledge
tags: []
---

# Knowledge

## Prisma: parent-vs-children filtering (the where footgun)

A top-level `where` decides WHETHER THE PARENT ROW returns. A `where` nested
inside a `select`/`include` filters WHICH CHILDREN come back — it does NOT gate
the parent.

```ts
// Account-level gate (parent) AND role-level filter (children), independently:
prisma.user.findUnique({
  where: { chatId, isActive: true },        // parent returns only if active
  select: {
    userRoles: {
      where: { revokedAt: null },           // include only non-revoked roles
      select: { role: true },               // (does NOT affect parent return)
    },
  },
});
```

Relation filters `some` / `none` / `every` in a top-level `where` also gate the
PARENT based on its children — not the children themselves:
- `some: { x }`  → parent has ≥1 child matching x
- `none: { x }`  → parent has 0 children matching x
- `every: { x }` → all children match x

Gotcha: to "return the user only if they have an active manager role" use
`some`; to "count active manager assignments" query the JOIN table directly
(`prisma.userRole.count({ where: {...} })`), not the user table.

## Prisma: extendedWhereUnique (GA since 4.5, default in v5+)

`findUnique`/`update`/`delete` can take NON-unique fields in `where` alongside
the unique one — the unique field identifies the row, the extra fields filter:

```ts
// Returns null if the row exists but isn't active (no need to fetch then check).
prisma.telegramUser.findUnique({ where: { chatId, isActive: true } });
```

## TypeScript: exhaustive unions with assertNever

Handling-then-returning each case narrows the union; by `default` it's `never`.
A `never`-typed param only accepts a value TS believes is `never`:

```ts
function assertNever(x: never): never {
  throw new Error(`Unhandled case: ${JSON.stringify(x)}`);
}

switch (result) {
  case "a": /* ... */ return;
  case "b": /* ... */ return;
  default: return assertNever(result); // add "c" to the union → COMPILE error here
}
```

Data counterpart: `Record<Union, T>` forces an entry for every member — add a
union member and the object won't compile until you handle it.

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
