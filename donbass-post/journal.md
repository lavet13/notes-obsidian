---
id: journal
aliases:
  - journal
tags: []
---

# Journal

## 2026-06-25 — RBAC: soft-delete revocation + notification backfill

### Covered
- **Soft-delete via `revokedAt` (UserRole):** revoking a role = stamping
  `revoked_at`, not deleting the row or disabling the account. Key insight:
  one flag, one meaning — `telegram_users.is_active` is account-level,
  `user_roles.revoked_at` is role-level. Don't overload one field for two jobs.
  Added `granted_at` too → each assignment has a [granted, revoked) lifespan
  (audit/history). `null` = active is the convention everywhere.
- **Two-gate read model:** `getUserPermissions/getUserRoles` gate at TWO levels —
  top-level `where: { isActive: true }` (account) AND nested
  `userRoles: { where: { revokedAt: null } }` (role). Parent-vs-children footgun
  again: top-level `where` decides if the user returns; nested `where` filters
  which roles count. `some/none/every` decide the PARENT, not the children.
- **Status-token return + exhaustive switch:** services return neutral tokens
  (`"revoked" | "not_a_manager" | ...`), handlers map tokens → localized UI.
  `assertNever(x: never)` in the `default` makes a missed case a COMPILE error —
  the compiler becomes the "did I handle everything" checklist. Caught a real
  typo this way (`already_active_manager` vs `already_manager`).
- **Staged data migration (notification prefs):** backfill → migrate writes →
  migrate reads → drop old. Never swap reads first: the new table is empty until
  backfilled, so reading it early = silent zero-notifications. Backfill is
  gated bootstrap-only (run only when new table empty) to avoid resurrecting
  runtime-removed rows from the frozen old table — same lesson as MANAGER_CHAT_IDS.

### Changed
- Migration `add_lifecycle_timestamps_to_user_roles`: added `revoked_at`
  (nullable) + `granted_at` (default now). Additive, verified, applied in prod.
- `removeManager`: revokes the manager UserRole (`updateMany` + `revokedAt: null`
  guard to preserve original timestamp); returns a status token. No longer
  touches `is_active`. Old `HACK:` overload comment resolved.
- `addManager`: find-or-create user (const + `??`, no mutable let), then branch
  fresh / reactivate / already-manager / role-not-found. Create-only-when-fresh
  (no surprise contact-info overwrite). All DB calls go through `tx`.
- Seed: bootstrap gate now counts `userRole` (active manager assignments) not
  users; added gated backfill old→new notification prefs (mapping
  `manager.telegramUserId → userId`). Verified: old=1, new=1.
- Migrated `getAllManagers` off the legacy Manager table onto UserRole.

### Open threads / resume from
1. **Notification migration step 2 (writes) — IN PROGRESS:** `setManagerPreferences`
   rewired to new table (pending the fix above); still TODO: the append/remove
   preference commands write to the old table.
2. **Step 3 (reads):** `getManagersForNotification`, `getManagerNotifications`,
   `isManagerSubscribed`, `getAllPreferences` still read the OLD table — migrate
   to `NotificationPreferences`. Rename during the notifications/ folder move
   (`getRecipientsForType`, `getUserSubscriptions`, `isSubscribed`,
   `getAllSubscriptions`, `setUserSubscriptions`).
3. **Extract `getManagerRole()`** — the verbatim `role.findUnique({name: MANAGER})`
   now repeats 3×. Extract just the lookup (not the differently-shaped guards),
   during the notifications split.
4. Then: Step 7 (drop legacy `Manager` + `ManagerNotificationPreferences`).

### Commit message (soft-delete work, ready once the FIX above lands)
feat(tg-bot): soft-delete role revocation via revoked_at

Revoke roles by stamping user_roles.revoked_at instead of flipping the
account-level is_active flag — separating account enablement from role
assignment. Add granted_at for assignment lifespan/audit.

- Migration: add revoked_at (nullable) + granted_at to user_roles
- removeManager: revoke the manager UserRole, preserve original timestamp,
  return status token; stop touching is_active
- addManager: find-or-create user, branch fresh/reactivate/already/role-missing
- read paths: gate on account is_active (top-level) AND role revoked_at (nested)
- seed: backfill notification prefs old→new (gated bootstrap-only);
  getAllManagers now reads UserRole, not the legacy Manager table
