---
id: forms-knowledge
aliases:
  - forms-knowledge
tags: []
---

# Forms Knowledge

Browser-form facts that keep biting the `/api/notify` work: what a form actually
sends, and why "the schema is wrong" is usually "I modelled the wrong thing."

## The producer is the mask + submit handler — never the placeholder/label

When you write a server schema for a form's payload, the authoritative source is
the code that SHAPES and SENDS the value, not the human-facing hint next to it.

```html
<!-- placeholder LIES: it says one thing, the mask overwrites it at runtime -->
<input name="pickupTime" placeholder="Время для забора с Х часов до Х часов">
```
```js
Inputmask({ mask: "99:99 - 99:99", placeholder: "__:__ - __:__" }).mask(el);
// wire value is "08:00 - 18:00" — a dash, no words. Model THIS, not the placeholder.
```

> Real outage (07-18): the bot's `pickupTime` regex matched Cyrillic `с ЧЧ:ММ до ЧЧ:ММ`
> — modelled on the placeholder text — while the mask emits `HH:MM - HH:MM`. 100% of
> payloads 400'd. **Order of authority: mask + submit handler > the value on the wire >
> the label. The placeholder is decoration.** Same class as modelling on `apps/web` when
> the real producer is the old PHP site.

## `hidden ≠ absent`: only `disabled` removes a control from FormData

```js
// A field hidden with display:none (or inside a hidden ancestor) STILL submits.
el.style.display = 'none';   // still in FormData, still carries its value
el.disabled = true;          // THIS removes it from FormData
```

> Consequence: a two-mode toggle that only hides the inactive field leaks stale input.
> Select a pickup point (`pointTo=5`), switch to address mode, type an address — nothing
> cleared `pointTo`, so BOTH ship. A presence-based / XOR check then 400s or guesses wrong.
> Visibility is cosmetic; `disabled` and value-clearing are the only things that change the
> payload.

> **`readonly` submits; `disabled` doesn't.** For a COMPUTED field that must still reach
> the payload (e.g. cubicMeter derived from dimensions), lock it with `readonly`, never
> `disabled` — disabled drops it from FormData → missing from a DOM-built payload → server
> 400 on a required field. (Verified: jsdom FormData excludes disabled, keeps readonly.)
> React can use `disabled` freely because it builds the payload from form STATE, not the DOM;
> a FormData-derived payload can't. Style an OWNED class (`.is-computed`), not `input[readonly]`
> — don't assume the lib reflects readonly to the DOM attribute.

## Clear-on-toggle: fix the leak at the source, not with a cross-field refine

A cross-field refine (`exactly one of A/B present`) and "guess the mode by which field is
present" are the SAME strategy — both break the instant both fields can be present. That's
unfixable in the schema (it only sees what's sent). Fix it in the FORM: on toggle, clear the
value of the now-hidden field.

```js
// reuse the visibility computation you already have — it names the hidden fields
const clearHiddenFieldValues = (form, fieldNames) => {
  fieldNames.forEach((name) => {
    const field = form.querySelector(`[name="${name}"]`);
    if (field) field.value = '';   // <select> value='' resets to its empty option
  });
};
```

> Once the form guarantees exactly one of the pair is populated, the empty one is dropped by
> the payload filter (`length > 0` / `> 0`), presence-guessing is unambiguous, and the XOR
> refine becomes a harmless backstop instead of a landmine.
>
> **Clearing keeps the field in FormData as `""`** (unlike `disabled`, which removes it).
> So a per-field "skip validation while hidden" guard (`shouldValidateField`) is STILL
> needed — clearing fixes the payload LEAK, not the validation-skip. Don't delete both.
>
> Checkboxes need `.checked = false`, not `.value = ''`.

## The discriminator may already be on the wire

A named radio (`name="address-option"` → `"select"|"input"`) rides in FormData and survives
a `length > 0` payload filter. So the form often TELLS you the mode explicitly — before you
reconstruct it by inference. A non-strict Zod object silently strips the unknown key, so it's
easy to throw away a fact you were handed. (Here we can't cleanly carry it — the primary order
POSTs to a separate backend that may reject the extra field — so we guess by presence instead.
But check for the discriminator before inventing one.)

## Re-encode the mask's guarantee on the server (anchor the regex)

The server has no input mask, so a regex that assumed one must re-state the guarantee — and
anchor it, or it matches garbage around a valid core.

```ts
// submit-time validator gets the FINISHED value→ anchor it
/^(\d{2}):(\d{2})\s*-\s*(\d{2}):(\d{2})$/     // bot + PHP submit check
// a DURING-TYPING matcher (isAllowed, \d{0,2}) must stay UNANCHORED & lenient — different job
```

> Two different functions with two different jobs: the on-submit validator anchors; the
> while-typing matcher stays permissive. Anchoring the wrong one breaks input. (Same principle
> as `innSchema` re-encoding "10–12 digits" because the server can't assume the mask.)

## Notify boundary: parse for shape, don't police content

`/api/notify` fires AFTER the real order is committed to workplace-post.ru and after the user
is told it succeeded. A schema rejection there can't prevent a bad order — it only LOSES a
manager notification. So parse enough to interpolate/narrow safely; don't enforce business
rules (cross-field XOR, enums on display-only values) that can 400 legit traffic.

> Caveat: the endpoint is publicly reachable, so keep enough validation that a direct console
> caller can't crash it or inject garbage into the message — format checks stay, they're
> hardening. The line is: shape = keep; content-policing on a value you only interpolate = drop.

## Rendering server errors: clear ALL, render one span per field

For a legacy form that relies on the server (not client JS) for validation, the two
functions that avoid the resubmit-duplication trap:

```js
function clearAllErrors(form) {
  form.querySelectorAll(".form-error-message").forEach((el) => el.remove()); // ALL — not one sibling
  form.querySelectorAll("[error]").forEach((el) => el.removeAttribute("error"));
}
function renderServerErrors(form, messages) {           // messages: { "sender.surnameSender": "…" }
  clearAllErrors(form);
  for (const [path, msg] of Object.entries(messages)) {
    const name = path.split(".").pop();                 // last segment; NOT /\.\w+/ (breaks on .0.)
    const field = form.querySelector(`[name="${name}"]`);
    if (!field) continue;
    field.setAttribute("error", "");
    const span = document.createElement("span");
    span.className = "form-error-message";
    span.textContent = Array.isArray(msg) ? msg.join("; ") : msg;   // ONE span per field
    (field.closest(".select-wrapper") ?? field).after(span);
  }
}
```

> The bug this replaces: clearing via `input.nextElementSibling` removes only the FIRST
> sibling, so a field with two error spans keeps the second → duplicates grow on every
> resubmit. Clearing by `querySelectorAll` (all of them) + one joined span per field
> removes both the leak and the duplication.
