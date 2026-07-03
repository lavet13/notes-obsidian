---
id: prisma-knowledge
aliases:
  - prisma-knowledge
tags: []
---

# Prisma-Knowledge

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

## Prisma: BigInt fields need BigInt() in where clauses

If a column is `BigInt` (e.g. Telegram chatId), a raw `number` in `where` won't
match — silently returns nothing, no error.

```ts
// WRONG — number where a BigInt is expected; matches nothing
where: { chatId: chatId }
// RIGHT
where: { chatId: BigInt(chatId) }
```
Symptom: a query that should find a row returns null/[] for no obvious reason.
First thing to check when a chatId/BigInt lookup "can't find" an existing row.
