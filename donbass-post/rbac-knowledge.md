---
id: rbac-knowledge
aliases:
  - rbac-knowledge
tags: []
---

# RBAC Knowledge

## Permission checks vs role-identity checks

Check "what can you do" (permission), not "who are you" (role identity).
`userHasPermission(user, "users:manage")` instead of `isAdmin(user)`. Roles are
just bundles of permissions; gating on a permission means adding a new role or
reshuffling permissions does not require touching every call site. Role-identity
checks (`isRootAdmin`, `isActiveManager`) scattered through handlers are the
thing to remove - they hardcode the role graph into business logic.

## Flat roles, many permissions, multiple roles per user

No role hierarchy. A user can hold several roles; each role bundles permissions;
the user's effective permissions are the union. Avoids the classic "admin
inherits from manager inherits from user" inheritance tangle - flat + union is
simpler to reason about and to query.

## The wildcard as a seed-time convenience, not a stored row

A root role can be declared with a `"*"` wildcard permission AT SEED TIME, but
it is expanded into concrete permission rows in storage - no literal `"*"` row
exists. Why: permission checks stay uniform (`has("users:manage")`), with no
special `|| includes("*")` branch anywhere in the hot path. The wildcard is a
declaration convenience; the database holds only concrete permissions.

## Permissions should be coarse (capability-grouped), not 1:1 per command

Group by capability (`users:manage`, `bot:view-status`), not one permission per
command. Too-fine permissions become a maintenance burden with no real access
benefit. Rule of thumb: a permission is a capability a role either has or does
not; commands are just the surface that checks it.

## Receiving a notification is a subscription, not a permission

Do not model "can receive notification X" as a permission. Receiving is a
SUBSCRIPTION - the presence of a preference row IS the signal. Permissions gate
actions ("what you may do"); subscriptions express opt-in state ("what you want
to get"). Conflating them puts opt-in data in the permission system where it
does not belong.

## Account-level vs role-level enablement (do not overload one flag)

Two separate concerns, two separate fields:

- `is_active` on the user/account = account-level enablement (the whole user).
- `revoked_at` on the role assignment = role-level (one specific role).
  Revoking a role must stamp `revoked_at`, NOT flip `is_active` - otherwise a
  fired manager also becomes a disabled account (and loses unrelated roles /
  client features). One flag, one meaning.

## Two-gate reads

A permission/role read must gate at BOTH levels:

- top-level `where: { isActive: true }` - does the ACCOUNT return at all?
- nested `userRoles: { where: { revokedAt: null } }` - which roles count?
  Miss the account gate and a deactivated user keeps permissions; miss the role
  gate and a revoked role still grants access. Both gates, always.

## Env vars are a bootstrap/recovery mechanism, not a runtime check

`ROOT_ADMIN_CHAT_ID` / `MANAGER_CHAT_IDS` seed initial access, but once runtime
management exists, authority lives at RUNTIME. Seed from env only when zero
records exist (gated bootstrap). If env re-asserted on every deploy, a runtime
removal would be undone on the next deploy. Keep the env bootstrap as a recovery
hatch (so you cannot lock yourself out) but never let it override runtime state.

## Lockout safety

When removing role-identity checks from the hot path, keep a recovery mechanism:
the env-based root bootstrap in the seed. The real invariant behind
"cannot remove yourself" is "do not leave the system with zero people who can
manage it" - a count check ("would this leave zero active admins?"), which is
environment-agnostic, beats a "is this self?" check gated on NODE_ENV.

## In-memory permission cache with explicit invalidation

Permission/role lookups hit the DB on every check, so cache them (a Map with a
short TTL, e.g. 60s). Critical: invalidate the cache on any write that changes a
user's access (add/remove role) - `invalidateUser(chatId)`. A cache without
invalidation serves stale permissions after a revoke, which is a security bug,
not just a staleness bug.

## Status-token returns drive exhaustive handling

Services return neutral status tokens (`"revoked" | "not_a_manager" |
"user_not_found" | ...`); handlers map tokens to localized UI. The token union
pairs with `assertNever` in the handler's switch default, so adding a new
outcome is a compile error until the UI handles it. Keeps domain logic free of
presentation and makes "did I handle every case" a compiler guarantee.

## Guard the invariant, not a proxy for it

`NODE_ENV === "production" && ctx.from?.id === chatId` ("can't remove yourself") was a PROXY —
wrong in both directions: it blocked a harmless self-removal when 10 managers exist, and it
never stopped an admin removing the LAST manager (someone else) → zero managers → notifications
silently go nowhere. Name the real rule and check THAT.

```typescript
// "Would this leave zero active managers?" — a DATA invariant, so it lives in the service,
// not the handler. Count + write in ONE transaction: it's check-then-act, same race shape as
// a read-modify-write, so the count must not be able to go stale before the update.
const result = await prisma.$transaction(async (tx): Promise<RemoveManagerResult> => {
  const user = await tx.telegramUser.findUnique({ where: { chatId: BigInt(chatId) }, select: { id: true } });
  if (!user) return "user_not_found";

  const assignment = await tx.userRole.findFirst({
    where: { userId: user.id, revokedAt: null, user: { isActive: true }, role: { name: Roles.MANAGER } },
    select: { roleId: true },   // also gives the compound-key half for the update
  });
  if (!assignment) return "not_a_manager";

  // count active managers OTHER than the target
  const others = await tx.userRole.count({
    where: { revokedAt: null, role: { name: Roles.MANAGER }, user: { isActive: true, id: { not: user.id } } },
  });
  if (others === 0) return "last_manager";

  await tx.userRole.update({
    where: { userId_roleId: { userId: user.id, roleId: assignment.roleId } },
    data: { revokedAt: new Date() },
  });
  return "revoked";
});
if (result === "revoked") invalidateUser(BigInt(chatId));   // cache work only on a real change
```

> Keep the definition of "active manager" IDENTICAL across every query
> (`revokedAt: null` + `role.name` + `user.isActive`). If the guard and the count disagree
> about what they're counting, the invariant rots.
> Adding a token to the result union + `assertNever` = the compiler forces the new handler branch.
