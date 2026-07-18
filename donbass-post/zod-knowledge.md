---
id: zod-knowledge
aliases:
  - zod-knowledge
tags: []
---

# Zod Knowledge

Env-validation and input-cleaning patterns. Zod is the "validate at the boundary,
fail fast with a clear message" tool (Pydantic is Python's equivalent).

## preprocess vs transform (the core distinction)

They look similar but run at opposite ends of validation:

- `preprocess(fn, schema)` runs BEFORE validation. Input type is `unknown` (it
  hasn't been validated yet). Use it to CLEAN raw input into the shape the schema
  expects.
- `.transform(fn)` runs AFTER validation. Input is the already-validated value.
  Use it to RESHAPE a value you've confirmed is valid.

```ts
// CLEAN before validating -> preprocess
z.preprocess((val) => (val === "" ? undefined : val), z.string().optional());

// RESHAPE after validating -> transform
z.string().transform((s) => s.trim().toLowerCase());
```

Rule of thumb: cleaning messy input = preprocess; deriving a new value from a
valid one = transform.

## The empty-env-string footgun

An empty env var (`FOO=`) arrives as the empty string `""`, not `undefined`.
This breaks the naive "optional coerced number":

```ts
// BUG: an empty secret becomes 0, not undefined.
z.coerce.number().optional();
// coerce runs FIRST: Number("") === 0. So "" -> 0 before .optional() ever sees it,
// and .optional() only fires on `undefined`, which never arrives.
```

The fix is to map blank -> undefined BEFORE coercion, restoring the meaning of
"absent" so `.optional()` / `.refine()` behave:

```ts
// Generic helper: blank/whitespace string -> undefined, then run the real schema.
function emptyAsUndefined<T extends z.ZodType>(schema: T) {
  return z.preprocess((val) => {
    if (typeof val === "string" && val.trim() === "") return undefined;
    return val;
  }, schema);
}

// Usage — now an empty env var is treated as unset, not coerced to 0:
TELEGRAM_PROXY: emptyAsUndefined(z.string().url().optional()),
ROOT_ADMIN_CHAT_ID: emptyAsUndefined(z.coerce.number().optional()),
```

Where the helper lives: keep it in `env.ts` (narrowest owner) until a second
file needs it — then promote it to a shared util.

## transform().pipe() — the other way to clean-then-validate

`preprocess` isn't the only option. You can `.transform()` a cleaner and `.pipe()`
the result into a real schema — the transform's OUTPUT becomes the pipe's INPUT
(like `cat | grep`):

```ts
z.unknown()
  .transform((val) => (val === "" ? undefined : val)) // clean
  .pipe(z.coerce.number().optional()); // validate the cleaned value
```

GOTCHA: a block-body arrow with no `return` returns `undefined`. In a transform
that must pass values through, you MUST explicitly return the pass-through case:

```ts
.transform((val) => {
  if (val === "") return undefined;
  return val;              // <- required, or every non-empty value becomes undefined
})
```

When to use which: `preprocess` reads clearer for a single cleaning step;
`transform().pipe()` shines when BOTH stages are real schemas (validate, reshape,
validate again).

## Practical: validating env at the boundary

The pattern that ties it together — one schema, parsed once at startup, so the
rest of the app reads typed, validated values from `env` and never touches
`process.env` directly:

```ts
const envSchema = z.object({
  TELEGRAM_BOT_TOKEN: z.string().min(1),
  NODE_ENV: z.enum(["development", "production"]).default("development"),
  ROOT_ADMIN_CHAT_ID: emptyAsUndefined(z.coerce.number().optional()),
  TELEGRAM_PROXY: emptyAsUndefined(z.string().url().optional()),
});

const parsed = envSchema.safeParse(process.env);
if (!parsed.success) {
  console.error("❌ Invalid environment variables:", parsed.error.format());
  process.exit(1); // fail fast, loud, at startup — not deep in a handler later
}
export const env = parsed.data;
```

`safeParse` returns `{ success, data | error }` instead of throwing — lets you
print a clear message and `process.exit(1)` rather than surfacing a raw stack
trace. Fail at the boundary, once, so nothing downstream has to re-check.

## Strict scalar parse: `z.coerce.number().int()`

For a whole-token numeric input (an id from a command string) you want STRICT parsing —
reject trailing junk and floats, not truncate them like `parseInt`.

```typescript
const ChatId = z.coerce.number().int(); // coerce = Number(raw); .int() rejects NaN AND floats
export function parseChatId(raw: string): number | null {
  const r = ChatId.safeParse(raw); // { success:true, data } | { success:false, error }
  return r.success ? r.data : null;
}
// '123abc' -> Number -> NaN -> fail;  '12.9' -> 12.9 -> .int() fail;  '-100' -> ok.
// vs parseInt('123abc')=123, parseInt('12.9')=12 (lenient truncation — wrong for a full id).
```

## Cross-field rules: `.refine()` on the object

```typescript
z.object({ sender: A.optional(), companySender: B.optional() })
  .refine(
    (d) => !!d.sender !== !!d.companySender, // XOR: exactly one
    { error: "…", path: ["sender"] },
  ) // path attaches the issue to a field
  .refine(
    (d) => !(d.customer && d.companyCustomer), // at most one (optional but exclusive)
    { error: "…", path: ["customer"] },
  );
```

> `.superRefine((d, ctx) => ctx.addIssue({...}))` when one check must raise several/targeted issues.
> **Zod v4:** an object-level `.refine()`/`.superRefine()` runs EVEN WHEN a field fails to
> parse — it receives the parsed-so-far data (a field that failed its own `.min()` still passes
> its raw value through), so cross-field + field errors surface together. Zod v3 GATED it (refine
> ran only if the whole object parsed). Verified on 4.4.3. Don't assume a refine is dormant just
> because a field is failing.

## `.default()` splits INPUT type from OUTPUT type

```typescript
timestamp: z.iso.datetime().default(() => new Date().toISOString()),
// INPUT: optional — client may omit.  OUTPUT: required — z.infer types it `string`.
```

> Use this instead of dropping `.optional()` when you want certainty downstream WITHOUT
> forcing the client to send the field (dropping .optional() changes the contract → 400s).

## Modelling "either A or B" payloads

Prefer a FLAT object with optional sibling keys + `.refine()` over `z.union` / intersections:

- `A.and(B)` returns a `ZodIntersection` → loses `.pick`/`.omit`/`.extend` (docs: prefer `A.extend(B)`).
- Union errors stack every branch's failures; refine gives one message at a `path`.
- `.optional()` on a union applies to the VALUE — at the top level the object is never undefined,
  so the union still runs. Optionality belongs on the KEY.
- `z.discriminatedUnion("type", …)` is better _when the wire payload carries a literal discriminator_.
  Discriminating by "which key is present" is not that — use flat + refine.

## "Stricter than the client" is only valid as a SUPERSET

```typescript
// WRONG here: form sends "Третье лицо(Заказчик)"; enum lists "Третье лицо" → disjoint → 400.
shippingPayment: z.string().min(1),  // value is display-only (interpolated in the formatter)
```

> Stricter-than-client is right only when your accepted set is a SUPERSET of what the client can
> send AND something downstream depends on the narrowing. For a value you only ever interpolate,
> an enum buys nothing and can only 400 when the label drifts. `z.infer` narrowing isn't worth a
> production outage on a display string.

## DRY the repeated field rules

```typescript
const text3to50 = (required: string, range: string) =>
  z
    .string({ error: required })
    .trim()
    .min(1, required)
    .min(3, range)
    .max(50, range);
const innSchema = z
  .string()
  .regex(/^\d{10,12}$/, "ИНН должен содержать от 10 до 12 цифр");
// ^ the server can't rely on an input mask, so encode the mask's rule (digits, max 12) in the regex.
```

## Phone validation on the server

```typescript
import { isPossiblePhoneNumber } from "libphonenumber-js"; // NOT react-phone-number-input (React UI lib)
const phoneSchema = z
  .string({ error: "Заполните телефон!" })
  .refine((v) => isPossiblePhoneNumber(v), {
    error: "Проверьте правильно ли ввели номер телефона!",
  });
```

> `isPossiblePhoneNumber` = length plausibility only (lenient, any country).
> `isValidPhoneNumber` = checks real number ranges. Pass a country for stricter: `(v, "RU")`.
> `{ defaultCountry: "RU" }` (object form) lets bare national formats — `8…`, `9…`, `8 (900)…` —
> parse; WITHOUT it only `+`-prefixed E.164 passes. So it's MORE permissive for local users, not
> stricter. `isPossiblePhoneNumber` = length plausibility; `isValidPhoneNumber` = real ranges.

## Modelling "either A or B" payloads — match the KEY structure

The right tool depends on how the producer shapes it. Read the producer, then pick:

**One key, two shapes** → `z.union` on the VALUE. The XOR is inherent; no refines needed.

```typescript
sender: z.union([SenderIndividual, SenderCompany]),   // { sender: <either shape> }
customer: z.union([A, B]).optional(),                 // .optional() works — the VALUE can be undefined
```

**Sibling keys, at most one present** → flat object + `.refine()`. `.optional()` on a union
would apply to the whole object (never undefined), so optionality must live on each KEY.

```typescript
z.object({ sender: A.optional(), companySender: B.optional() }).refine(
  (d) => !!d.sender !== !!d.companySender,
  { error: "…", path: ["sender"] },
);
```

**A literal discriminator on the wire** (`{ type: "individual", … }`) → `z.discriminatedUnion("type", […])`.
Best errors, but only if the payload actually carries the tag.

> Tradeoff of unions: real strictness. A value must match one branch EXACTLY or you get a
> confusing "invalid union" 400 — no partial-credit like a bag of optionals gives you.
