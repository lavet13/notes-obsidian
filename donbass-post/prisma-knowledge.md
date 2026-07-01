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

---

# Anki's cards

Card 1 - parent vs children where (recall)

Front: Prisma - what is the difference between a top-level where and a where nested inside a select/include?
Back: Top-level where decides WHETHER THE PARENT ROW returns. A nested where filters WHICH CHILDREN come back - it does NOT gate the parent.

Card 2 - parent vs children where (application)

Front: Prisma - you want a query to return a user only if they have an active (non-revoked) manager role, AND to include only their non-revoked roles. How do you structure the two conditions?
Back: Two independent conditions: a top-level relation filter (userRoles: { some: { revokedAt: null, role: { name: MANAGER } } }) gates the PARENT; a nested where inside the select (userRoles: { where: { revokedAt: null } }) filters the CHILDREN. One decides if the user returns, the other decides which roles you see.

Card 3 - some/none/every (recall)

Front: Prisma - what do the relation filters some, none, and every do in a top-level where?
Back: They gate the PARENT based on its children (not the children themselves). some: {x} = parent has >=1 child matching x; none: {x} = parent has 0 matching; every: {x} = all children match x.

Card 4 - counting children (application)

Front: Prisma - you need to count active manager assignments. Do you query the user table with a relation filter, or the join table directly?
Back: Query the JOIN table directly: prisma.userRole.count({ where: { revokedAt: null, role: { name: MANAGER }, user: { isActive: true } } }). Use some/none/every when you want to filter PARENT rows by their children; use a direct count on the join table when the count itself is the answer.

Card 5 - extendedWhereUnique (recall)

Front: Prisma - what does extendedWhereUnique allow (GA since 4.5, default in v5+)?
Back: findUnique/update/delete can take NON-unique fields in where alongside the unique one. The unique field identifies the row; the extra fields filter it.

Card 6 - extendedWhereUnique (application)

Front: Prisma - you want to fetch a user by chatId but return null if their account is inactive, without a separate check. How?
Back: findUnique({ where: { chatId: BigInt(chatId), isActive: true } }) - the unique field (chatId) identifies the row, the non-unique field (isActive) filters it, so an inactive row returns null. This is extendedWhereUnique.

Card 7 - BigInt in where (recall)

Front: Prisma - what happens if you pass a raw JS number into a where clause for a BigInt column?
Back: It silently matches nothing - no error, no warning. You get null/[] as if the row does not exist.

Card 8 - BigInt in where (application)

Front: Prisma - a chatId lookup returns null for a row you KNOW exists in the database. What is the first thing to check?
Back: Whether the column is BigInt and you passed a raw number. Wrap it: where: { chatId: BigInt(chatId) }. A number-vs-BigInt mismatch matches nothing silently - it is the classic "can't find an existing row" cause.
