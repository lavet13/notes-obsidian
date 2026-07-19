---
id: journal
aliases:
  - journal
tags: []
---

# Journal

## 2026-07-19 — online-pickup verified in prod; old-form dimensions fix; pick-up-point investigation opened

**online-pickup-rf: CLOSED.** Deployed the bot `types.ts` fix. Prod-verified both modes + the worst case (filled customer then collapsed; filled pickupAddressRecipient then switched to pointTo — all in one submit). Payload dump confirms clear-on-toggle holds: `pointTo: 49` present, `pickupAddressRecipient` and all four customer fields absent from the payload. `shippingPayment: "Третье лицо(Заказчик)"` now accepted via `.string().min(1)`. Six-day notification outage over.

**Old PHP form — dimensions/cubicMeter (the ⚠️ image bug):** длина/ширина/высота were marked `isFilled`-required in `validationRules`, firing "Обязательно" even when cubicMeter was filled. Fix = DELETE the three dimension rules — they're computation inputs, not required fields (the form's own copy says so, and `apps/web` gives them no validators). cubicMeter's `> 0` rule is the single volume invariant; every path (type cubicMeter / fill dims / fill nothing) is covered by validating only the canonical field. Removing rules fixed it; a cross-field skip would've been the complex wrong answer.

**cubicMeter lock — `disabled` would've silently re-opened a 400.** Wanted: lock cubicMeter when all 3 dims filled. `disabled` EXCLUDES a control from FormData (verified via jsdom), and this form builds `data` from FormData entries → a disabled cubicMeter → `data.cubicMeter` undefined → dropped from payload → bot's `positive()` 400s. `readonly` SUBMITS. Used `anCubicMeter.options.readOnly(allDims)` in one `syncCubicMeterLock()` (single source of truth; dropped a redundant inner `readOnly(true)`). Visual: toggle an owned `is-computed` class (grey + `not-allowed`) + a `title` explaining why/how-to-unlock — NOT `input[readonly]` CSS (don't couple to whether autoNumeric reflects readonly to the DOM attribute). React gets away with `disabled` because it builds the payload from form STATE, not the DOM — same keyword, opposite result.

**Resume:** investigate the OTHER notify endpoint — `pick-up-point-delivery-order` — against its old PHP producer (`zayavka-nazaborDNR-new.php`). Never tested since the zod migration. Schema is DEEPLY NESTED (`{ sender: I|C, recipient: I|C, customer: (I|C)?, cargoData }`) unlike online-pickup's flat shape — so the decisive unknown is whether the producer POSTs nested or flat. Need the producer's payload dump or its transform JS. Risk map below.

## 2026-07-18 — online-pickup-rf notify: total outage diagnosed + fixed

**The outage.** Zero online-pickup-rf notifications since the 07-12 zod migration. Root cause: the bot's `pickupTime` regex expected Cyrillic `с ЧЧ:ММ до ЧЧ:ММ`, but the producer (old PHP form) masks the field with Inputmask `"99:99 - 99:99"` → emits `"08:00 - 18:00"`. Every payload 400'd on pickupTime first. **Modelled the label, not the machine** — the input's `placeholder` reads "с Х часов до Х часов" but Inputmask overwrites it at runtime; the mask + submit handler are the producer, never the placeholder. Same class as modelling on apps/web (07-12): a plausible neighbour that reads like the truth.

**Second dead-stop 400:** `shippingPayment` enum vs the form's `"Третье лицо(Заказчик)"` — disjoint set, not "stricter." Fixed to `.string().min(1)` (value is pure interpolation). Corrected the zod-knowledge "stricter than client" note, which used this exact field as its example.

**Correction (evidence = my own probe):** claimed v3's "refine runs only if the object parses." **Zod v4 changed this** — object refines run even when fields fail (verified 4.4.3). So the XOR/customer refines HAVE run in prod; a real rejection log WOULD show cross-field issues. Re-examined and corrected mid-session.

**Bugs traced (refines are live, not dormant):**

- XOR (C): reachable — select a point, toggle to address mode, type address; nothing clears `pointTo` → both ship → 400.
- Customer count (D): open, type one field, collapse, submit → `filled===1` → 400. `hidden ≠ absent`: only `disabled` drops a control from FormData.
- `phoneSchema`: `isPossiblePhoneNumber(v)` with no country rejects `8…`/`9…` → latent outage behind pickupTime. Fixed with `{ defaultCountry: "RU" }`.
- `validatePickupTime` destructure: `.map(Number).filter(isInteger)` drops the full-match slot, so the skip-comma eats a real group → `endMin` undefined. Fixed in all three copies (bot, php, react util).

**Design throughline:** `/api/notify` fires AFTER the order is committed to workplace-post.ru and after the user is told it succeeded → rejecting can only LOSE a notification, never prevent a bad order. Parse for shape, don't police content. Caveat (Ivan): the endpoint is publicly reachable, so keep `end<=start`/`<120` as hardening against direct callers — not removed.

**names/min-3 — left as-is (Ivan's call, correct):** workplace-post.ru enforces 3–50 and runs BEFORE notify in the submit sequence; a sub-3 name 400s there and never reaches the bot. Upstream gate ordering makes the bot's min-3 unreachable by legit traffic.

**Discriminator:** producer sends `address-option` on the wire, but the separate primary endpoint means we can't cleanly carry it → guess by presence. VALID only if the form guarantees exactly one of pointTo/pickupAddressRecipient present — i.e. clear-on-toggle. Schema stays; form changes.

**Resume / next step:**

1. DEPLOY the bot `types.ts` fix — the actual outage fix (pickupTime anchored + destructure + phone defaultCountry + shippingPayment). Notifications restore on deploy.
2. PHP clear-on-toggle (C+D): `clearHiddenFieldValues` helper + call in `updateAddressFieldsUI`; clear the 4 customer inputs in the collapse branch. `shouldValidateField` STAYS (clearing keeps fields in FormData as "").
3. react `lib/utils.ts` `validatePickupTime`: anchor the regex (not `isAllowed`).
4. VERIFY the both-repro against workplace-post.ru (DevTools Console+Network) before trusting the clear-on-toggle blast radius.

**Commit (bot, on deploy):**
​```
fix(notify): repair online-pickup-rf schema — total notification outage since 07-12

- pickupTime: match the producer's masked "HH:MM - HH:MM", not Cyrillic "с … до"
- pickupTime: fix skip-comma destructure (filter dropped full-match slot → endMin undefined)
- phone: accept RU national formats via { defaultCountry: "RU" }
- shippingPayment: z.string().min(1) — display-only value; enum was disjoint from the form
  ​```

## 2026-07-12 (cont.) — NODE_ENV gate → count invariant

**The gate was a proxy.** `NODE_ENV === "production" && ctx.from?.id === chatId` failed in both directions: too strict (blocked a harmless self-removal when other managers exist) and too weak (never stopped an admin removing the LAST manager — someone else — leaving zero, with notifications silently going nowhere). **Key insight: `NODE_ENV` is never the real rule.** It's a stand-in for "be careful where mistakes hurt" — but harm doesn't care about the environment, so the proxy is right by coincidence and wrong by construction.

Replaced with the real invariant — "would this leave zero active managers?" — in `removeManager` (service, not handler: it's a data rule). Count + write in one `$transaction`, since it's check-then-act (same race shape as the read-modify-write we fixed in `/appendpreference`). New `last_manager` token → `assertNever` forced the handler branch. Dropped `user_deactivated` (an inactive user can't hold a _counted_ active assignment, so `not_a_manager` covers it). Added `user: { isActive: true }` to the assignment lookup so all three queries share ONE definition of "active manager" — guard and counted set must agree or the invariant rots.

**Also fixed now rather than parked:** `setCommandsForChat`/`clearCommandsForChat` now have their own try/catch. The DB write is committed and is the source of truth; a menu push failing (chat not found, 429) must not report the add/remove as failed. `/addmanager 999999999` used to say "❌ ошибка" while the manager _was_ added. Also deleted the `result === "fresh" || "reactivated" || "already"` guard — it listed all three non-error outcomes, so it only _looked_ like a decision.

**Found: psql-knowledge.md documented a broken command.** `docker compose exec postgres psql -U "$POSTGRES_USER"` — double quotes mean the HOST shell expands it, but the var lives in the container via `env_file` → empty → `psql -U -d db` reads `-d` as the username. Fix: `sh -c 'psql -U "$POSTGRES_USER" …'` — single quotes defer expansion to the container. Note corrected.

**Testing note:** no second Telegram account needed — `/addmanager <fake chatId>` now works cleanly (scope push fails harmlessly). Sequence: remove-self as sole manager → ⛔ last_manager; add fake; remove-self → ✅; re-add self; delete fake. This is the first genuine candidate for an automated test if a test DB ever gets stood up (guards a catastrophic silent state, hard to reproduce by hand) — but needs real Postgres, so parked.

**Resume:** all non-parked bot items are done. Remaining is old-site JS bugs (fix when touching the PHP site), structural moves (`core/`, types.ts split), and CI chores (checkout@v5, knip).

## 2026-07-12 — /api/notify zod migration: shipped

Deployed. Verified against the pushed tree: unions in place, top-level XOR refines gone, formatters narrow with `in`, `handleNotify` live, `REGISTER_COMMANDS` fully removed from env.ts.

**The session's real lesson — I modelled the wrong client, twice.** `apps/web` looks like the frontend, but it POSTs to `workplace-post.ru` (co-worker's backend). `/api/notify`'s actual producer is the **old PHP site's jQuery**. Modelling on React gave: sibling `{sender} | {companySender}` keys (wrong — it's one `sender` key, two shapes), `pointFrom: number` (wrong — the producer resolves ids→display names before POSTing), and a "whatsapp casing typo fix" that was actually two producers legitimately differing (would have silently dropped the field — `.default(false)` hides it). **The old hand-written interface disagreed with me every time and was right every time.** Working code is evidence.

**Shape → tool:** one key with two shapes = `z.union` on the VALUE; the XOR is inherent, so all three hand-written refines evaporated. (Earlier note in zod-knowledge.md said "prefer flat+refine over unions" — that was derived from the wrong shape and has been CORRECTED in the file.) Formatters use `in` narrowing: same runtime test as the old truthiness checks, but the compiler understands it.

**`z.infer` as an audit:** tightening types produced 47 compile errors that were really a checklist — and surfaced two live prod bugs (formatter printed the SENDER's pointFrom as the recipient's pickup point; read `service.name`, which the wire never carried → `undefined`). The loose interface let the code lie.

**Old-site JS bugs found:** `getFormattedServices` had a block body with no `return` → `additionalService` never reached the bot at all (fixed); `indexOf(service.id)` on an object array → always -1; company customers silently dropped (payload gates `customer` on an individual-only field); recipient transform early-returns after `pointTo`, so `deliveryCompany` would ship as a raw id (latent — `pointTo` is never set today).

**Resume:** `NODE_ENV==="production"` self-removal gate → the real "would this leave zero active managers?" count invariant. Small, self-contained, environment-agnostic; also fixes single-user testing.

## 2026-07-12 — /api/notify zod migration: the pick-up-point-delivery schema

Started the parse-don't-validate migration. `ali-parcel-pickup` was already zod (`parseBody` + schema) — it's the template; the other two endpoints hand-roll checks.

**The big catch — the server was validating a payload that doesn't exist.** Its manual checks assumed `sender` with optional company fields inside (inclusive-OR: `if (!hasPhysical && !hasCompany)` errors only on _neither_, so both was legal). Read the frontend: `transformFormDataToPayload` branches on a `type` radio and emits **different keys** — `{sender} | {companySender}`, exclusive, differently shaped. Confirmed by `PickUpPointDeliveryOrderVariables` (optional sibling keys). So it IS real XOR, and the old code modelled a fiction. **Rule: model the payload the producer actually sends, verified by reading the client.**

**Design chosen:** one flat `z.object` with optional sibling keys + three `.refine()`s, NOT `z.union`/intersections. Why: intersections return `ZodIntersection` (loses `.pick`/`.omit`/`.extend` per zod docs), stack union errors, and can't express "customer optional" (`.optional()` on a union applies to the _value_, which is never undefined at the top level). Flat + refine matches the wire shape, gives per-party error `path`s, and `z.infer` reproduces the frontend type exactly.

**XOR in JS:** `!!a !== !!b` = exactly one. `!(a && b)` = at most one (customer: optional but exclusive). **Trap:** chaining `!!a !== !!b !== !!c` is _parity_ (true when an odd number are true), not "exactly one" — for N, count: `[a,b,c].filter(Boolean).length === 1`.

**Also:** `.default()` splits input from output type (client may omit, `z.infer` says required) — used for `timestamp`/`source`, since the frontend never sends them and making them required would 400 every order. `isPossiblePhoneNumber` from **libphonenumber-js**, not react-phone-number-input (React UI lib on a Node bot; the former is the real home). `z.enum` for `shippingPayment` — server is stricter than the client, which is right: the client isn't the security boundary.

**Resume:** finish the field-level validation port (helpers `text3to50` / `innSchema` / `positive` mirror the form's rules), then swap `routes/index.ts` to `parseBody(PickUpPointDeliverySchema, body)` — deletes ~80 lines — and delete the hand-written `PickUpPointDeliveryOrderPayload` interface in favour of `z.infer`. Then port `online-pickup-rf` (flat; its `requiredFields.filter(f => !payload[f])` also has the `!0`-is-falsy bug).

## 2026-07-11 (cont.) — reactive command registration (#3), 429 resolved

**#3 shipped:** deleted the per-manager boot loop; a manager's command scope is now set in the `/addmanager` handler (all three non-error outcomes — `already_manager` too, since the menu may have been cleared) and torn down in `/removemanager` on `revoked`. Boot only sets the static scopes now (public, all_private_chats, admin). Closes the parked menu-source-mismatch note (menus now flow from the same DB write-path).

- `setCommandsForChat` param widened `bot: Bot` → `api: Api` (it only ever used `bot.api`) — accept the narrowest dependency. Handlers pass `ctx.api`.
- Added symmetric `clearCommandsForChat`. **Key Telegram insight:** each `{scope, language_code}` pair is an independent list; `setCommandsForChat` writes 4 (LOCALES), so removal must `deleteMyCommands` per-language or the localized menus survive. Scope precedence: chat → all_private_chats → default, so deleting the chat scope falls back to public.

**#1 reverted:** the `REGISTER_COMMANDS` dev-gate was NOT kept — #3 removed the real amplifier, and gating dev hid useful signal (you want to see command registration while developing). Supersedes the earlier "Gated command registration" Done entry. (Check `REGISTER_COMMANDS` isn't left dead in env.ts.)

**#2 (hash-gated boot) — considered, YAGNI.** After #1+#3, the residual is ~a dozen fixed calls, prod-only, on rare restarts, under the limit. A persisted checksum trades cheap self-correcting re-pushes for a cache that can lie (DB hash desyncs from Telegram's real state → menu silently not restored). Not worth it at this scale.

**Guard lesson:** `if (!chatId)` was wrong — `parseChatId` signals failure only with `null`, but `!` also fires on `0`. Match the guard to the sentinel: `if (chatId === null)`.

**Handler vs service:** handler = the edge (has `ctx`/`api`, talks to Telegram); service = the core (Prisma only, no `ctx`). Telegram concerns (setting scopes) live in handlers; that's why `setCommandsForChat` isn't in `addManager`.

**Resume:** pick next — `/api/notify/*` zod boundary validation (parse-don't-validate, meatier), or the `NODE_ENV` self-removal gate → "would this leave zero active managers?" count invariant (quick win).

## 2026-07-11 (cont.) — post-review: userId threading, dead-code sweep, 429 gate

Reviewed the completed refactor. `resolveManagerCommand` now returns `userId` (rule-of-three cleared it — getAllManagers already needed the user row), so append/remove dropped their redundant per-command manager `findUnique`. `getAllManagers` → `{chatId, userId}[]`, threaded through `commands/index.ts`.

- **Dead code after extract-and-replace:** the atomic `count` refactor orphaned `isManagerSubscribed` (its only callers were append/remove). Deleted. Key insight: `yarn check-types` stays green on dead _exports_ — TS treats exported symbols as used-externally, so `noUnusedLocals` never flags them. Catch with grep or knip/ts-prune.
- **429 on boot:** `registerCommands` re-pushes all `setMyCommands` scopes every boot; tsx restarts on every save → cumulative rate-limit exhaustion (retry_after 841). Gated registration behind `NODE_ENV`/`REGISTER_COMMANDS` (fixed the `.default(true)`→`.default(false)` bug that made the gate a no-op). tsx watch = full process restart, NOT HMR — every startup side-effect re-runs.
- **config.managers is redundant + buggy:** `/status` counts managers from the env bootstrap list, not the DB. Cleanup = migrate `status.ts`/`server.ts` to `getAllManagers()`, drop `AppConfig.managers`, keep `MANAGER_CHAT_IDS` for seed only.

**Resume:** apply the managers-field cleanup + `REGISTER_COMMANDS` default fix; then `NODE_ENV` count-invariant gate, then zod for `/api/notify/*`.

## 2026-07-11 — Preference commands: relation-filter refactor, arg-parser extraction, concurrency fix

Resumed post-RBAC-migration cleanup. Started at "extract getManagerRole"; ended having _deleted_ it — the session's throughline was **the best refactor often removes the need, not the duplication**.

**getManagerRole — extract vs delete.** Counted the actual code, not the plan: the `role.findUnique({name})` was 2× not 3× (removeManager already filtered via the relation). That was the tell that the lookup itself was avoidable. Decided `role_not_found` was dead defensive code (role is seeded), dropped the distinction, went relation-filter everywhere. → `getManagerRole` deleted with zero callers.

- `addManager` reworked: `findFirst({ where:{ userId, role:{name} } })` for the lookup, `role:{connect:{name}}` on create, reactivation `update` via the fetched row's own `roleId` (UserRole has a compound `@@id`, no scalar id). `manager_role_not_found` token gone.

**Arg-parsing extraction → `commands/args.ts`.** `parseChatId` (pure; `Number.isInteger(Number(x))` — strict, unlike `parseInt`), `parseCommandArgs`, `resolveManagerCommand` (parse + authorize). Key insight: **return a discriminated-union Result, don't reply inside** — keeps I/O out and stays testable. Slug validation does NOT belong in the generic parser (it's domain knowledge, varies per command) — moved to the command layer as a `isNotificationSlug` **type guard** (`s is T` verifies-then-narrows; `as` only asserts and can lie). Fixed the `/setpreferences 123` clear-all bug for free (parser no longer demands a slug).

**Token → UI.** `subscriptionErrorReply` = exhaustive `switch` + `assertNever` (compile error when the union grows), extracted because the error arms were identical across append/remove/set.

**Concurrency.** `/appendpreference` + `/removepreference` did a read-modify-write across 3 calls → lost-update race. Fixed by shape: single-row atomic ops — `createMany({skipDuplicates})` / `deleteMany` — with the returned `count` as the changed/no-op signal. Kept `setManagerSubscriptions` for `/setpreferences` (absolute set). Insight banked: **read-to-decide = race, read-to-display = safe** (so the post-mutation "current subscriptions" re-fetch is fine).

**Also:** deploy now skips docs-only pushes (`!apps/telegram-bot/*.md` in the paths filter); version bumped minor. Side quests captured to knowledge files: GHA/glob/regex/BRE-ERE/anchoring, YAML (new `dev-env/yaml-knowledge.md`), Anki backup model (tsv = content, AnkiWeb/.colpkg = scheduling).

**Open / resume:** Ivan is about to paste his completed append/remove/set + `args.ts` for review (he reports backticks fixed, dead `findUnique` dropped in remove, guard collapse done). Next after review: `NODE_ENV` gate → count invariant; then zod for `/api/notify/*`.

**Commit (suggest 2–3 logical splits):**

```
refactor(tg-bot): relation-filter manager lookups, drop getManagerRole

Filter the MANAGER role via the relation everywhere (findFirst/connect-by-name);
addManager reactivates via the fetched row's roleId. Removes the standalone role
lookup and the manager_role_not_found / role_not_found branches (role is seeded).

feat(tg-bot): atomic preference add/remove, shared command arg parser

/append+/removepreference now do single-row createMany{skipDuplicates}/deleteMany
with count as the signal, killing the read-modify-write race. Extract
commands/args.ts (parseChatId, parseCommandArgs, resolveManagerCommand) +
isNotificationSlug guard; slug validation moved to the command layer.
Fixes /setpreferences <chatId> clear-all.

ci(tg-bot): skip deploy on docs-only changes (!apps/telegram-bot/*.md)
```

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
