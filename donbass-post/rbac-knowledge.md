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

# Anki's cards

Card 20 - two-gate reads (application)

Front: RBAC - a removed manager must lose access. Where do the two gates go in a permission-read query?
Back: Two levels: top-level where: { isActive: true } gates the ACCOUNT; nested userRoles: { where: { revokedAt: null } } gates the ROLE. Account-level and role-level enablement are separate concerns - do not overload one flag for both.

Card 21 - is_active vs revokedAt (recall)

Front: RBAC - what is the separation of concerns between telegram_users.is_active and user_roles.revoked_at?
Back: is_active is ACCOUNT-level enablement (the whole user). revoked_at is ROLE-level (one assignment). Revoking a manager role stamps revoked_at - it must NOT flip is_active, or a fired manager also becomes a disabled account.

Card 22 - status-token return pattern (recall)

Front: Architecture - why have a service return a neutral status token (e.g. "revoked" | "not_a_manager") instead of a formatted message?
Back: Separation of concerns: the service returns WHAT happened (a token), the handler maps the token to localized UI. Keeps domain logic free of presentation, and the token union pairs with assertNever for exhaustive handling.

Card 23 - env as bootstrap not runtime (recall)

Front: Architecture - what is the role of an env var like ROOT_ADMIN_CHAT_ID or MANAGER_CHAT_IDS once runtime management exists?
Back: It is a BOOTSTRAP / recovery mechanism, not a runtime check. Seed from env only when zero records exist (gated bootstrap). If it re-asserted on every deploy, runtime removals would be undone - authority must live at runtime.

Card 24 - const over let (recall)

Front: Refactoring - what does a let that is written exactly once usually indicate?
Back: A const waiting to happen. Prefer find-or-create with const x = found ?? await create(...) over a mutable let reassigned once - immutable, no | null to thread through.

Card 25 - business rules vs NODE_ENV (application)

Front: Design smell - you gate a business rule (e.g. "cannot remove yourself") on NODE_ENV === "production" to ease testing. What is wrong and what is the fix?
Back: You changed a business rule to fit testing, so the real rule never runs in dev - it can break unseen until prod. Fix: keep rules environment-agnostic. Replace "is this self?" with the true invariant ("would this leave zero active managers?") - a count check that works everywhere.

Card 26 - narrowest owner (recall)

Front: Project organization - where does a helper live: shared util, feature folder, or infrastructure?
Back: Narrowest owner until shared. Pure/stateless with no domain knowledge -> utils/. Touches one domain -> that feature folder. Shared by MULTIPLE features -> infrastructure (e.g. rbac/). Rule of three before extracting.

Card 27 - handler responsibility (recall)

Front: Architecture - what belongs in a handler (command/route/middleware) vs a service?
Back: A handler is a thin entry point: parse external input -> call a service -> format the reply. Logic lives in services. Test: "is this triggered by an external event (message, request)?" Yes -> handler.

Card 28 - filter vs subject (application)

Front: Organization - getManagersForType filters subscriptions by the manager role. Does it belong in the managers/ folder or notifications/?
Back: notifications/. The SUBSCRIPTION is the subject; the manager-role gate is just a filter. Classify by the primary concept, not by an incidental filter - which is also why the gate is easy to relax later for clients.
