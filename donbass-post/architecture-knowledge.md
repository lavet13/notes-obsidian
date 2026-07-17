---
id: architecture-knowledge
aliases: []
tags: []
---

# Architecture Knowledge

## Validate the payload the REAL producer sends

A schema is a claim about the wire. Verify it against the actual client — not the tidiest one.

> Real case: the bot's `/api/notify` client is an old PHP site's jQuery, NOT `apps/web`
> (which posts to a different backend entirely). Modelling on the React app produced a schema
> that would 400 every live request. Identify the producer before writing the schema.

## Working code is evidence

When existing code — especially two places independently agreeing — contradicts your model,
the prior is "they know something I don't," not "double typo."

> Real case: two endpoints used different casing (`whatsappSender` vs `whatsAppSender`).
> Looked like a bug; it was two producers. "Fixing" it would have silently dropped a field
> (a `.default(false)` makes the absence look intentional).

## Tightening types surfaces latent bugs

Replacing hand-written interfaces with schema-inferred types turns the compiler into an
exhaustive audit of everything that believed the old fiction.

> Real case: `z.infer` exposed a formatter printing the SENDER's pickup point as the
> RECIPIENT's, and reading a field the wire never carried (printing `undefined` in prod).
> The loose interface let the code lie; nothing could catch it.

## Best-effort side effects get their own try/catch

When a handler does a DURABLE write and then a side effect that can fail independently,
wrap the side effect separately. Otherwise the outer catch conflates "it didn't happen" with
"it happened but the follow-up didn't" — and you tell the user a lie about their own data.

```typescript
const result = await addManager({ chatId, … });   // committed — source of truth

// Best-effort: a menu push can fail (chat never started the bot, 429, network).
// That must NOT surface as "failed to add manager" — the manager WAS added.
try {
  await setCommandsForChat(ctx.api, chatId, publicCommands, managerCommands);
} catch (err) {
  console.error(`Command scope not set for ${chatId} (manager still added):`, err);
}
```
> Real symptom this fixes: `/addmanager 999999999` (a fake chatId, for testing) committed the
> row, then Telegram 400'd "chat not found", the outer catch fired, and the reply said
> "❌ Произошла ошибка" — while the manager existed. The DB write is the truth; the
> notification about it is not.
> Same shape already used in `sendToManagers`: a failed `notificationLog` write logs and
> continues rather than failing the send.
