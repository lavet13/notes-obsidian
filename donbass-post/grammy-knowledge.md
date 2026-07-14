---
id: grammy-knowledge
aliases: []
tags: []
---

# grammY / Telegram Commands Knowledge

## setMyCommands is a full overwrite, per {scope, language_code}
Each `{scope, language_code}` pair is an INDEPENDENT command list with its own 100-cmd limit.
`setMyCommands` REPLACES that pair's list entirely (no partial update; empty array = clear).
```typescript
// Setting a chat scope across languages = N independent writes:
for (const lang of [undefined, "ru", "uk", "en"])
  await api.setMyCommands(cmds, { scope: { type: "chat", chat_id }, language_code: lang });
```

## Removal must be symmetric across languages
`deleteMyCommands` also targets one `{scope, language_code}` pair. If you SET 4 language
variants, a single delete (no language_code) clears only the default one — localized menus
survive. Mirror the set:
```typescript
for (const lang of LOCALES)
  await api.deleteMyCommands({ scope: { type: "chat", chat_id }, language_code: lang });
```

## Scope precedence (private chat)
Telegram resolves the menu in order, most-specific first:
`chat` → `all_private_chats` → `default` (and within each, language-specific before language-agnostic).
> So deleting a chat-scoped list makes that chat FALL BACK to `all_private_chats`. If that's your
> public menu, an ex-manager cleanly reverts to public — no need to explicitly re-set public.

## Register reactively, not on every boot
Command scopes are server-side Telegram state that PERSIST across restarts. Re-pushing them every
boot is wasted calls (and a per-entity loop is an O(N) rate-limit amplifier). Set an entity's scope
in the lifecycle handler that changes it (`/addmanager` → set, `/removemanager` → clear); leave only
the static global scopes at boot. This also keeps menu state on the same DB write-path as everything else.
```typescript
// setCommandsForChat(api, chatId, ...groups) — accept `api`, not the whole `bot` (narrowest dep).
```
