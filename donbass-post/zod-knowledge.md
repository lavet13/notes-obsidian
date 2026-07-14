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
  .pipe(z.coerce.number().optional());                // validate the cleaned value
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
const ChatId = z.coerce.number().int();   // coerce = Number(raw); .int() rejects NaN AND floats
export function parseChatId(raw: string): number | null {
  const r = ChatId.safeParse(raw);        // { success:true, data } | { success:false, error }
  return r.success ? r.data : null;
}
// '123abc' -> Number -> NaN -> fail;  '12.9' -> 12.9 -> .int() fail;  '-100' -> ok.
// vs parseInt('123abc')=123, parseInt('12.9')=12 (lenient truncation — wrong for a full id).
```
