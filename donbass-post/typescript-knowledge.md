---
id: typescript-knowledge
aliases: []
tags: []
---

## TypeScript: exhaustive unions with assertNever

Handling-then-returning each case narrows the union; by `default` it's `never`.
A `never`-typed param only accepts a value TS believes is `never`:

```ts
function assertNever(x: never): never {
  throw new Error(`Unhandled case: ${JSON.stringify(x)}`);
}

switch (result) {
  case "a": /* ... */ return;
  case "b": /* ... */ return;
  default: return assertNever(result); // add "c" to the union → COMPILE error here
}
```

Data counterpart: `Record<Union, T>` forces an entry for every member — add a
union member and the object won't compile until you handle it.

```typescript
type Notification = "online-pickup-rf" | "ali-parcel-pickup" | "pick-up-point-delivery-order";

// A Record keyed by the union forces an entry for EVERY member. Add a 4th
// notification type and this object won't compile until you add its emoji.
const ICONS: Record<Notification, string> = {
  "online-pickup-rf": "📦",
  "ali-parcel-pickup": "🛍",
  "pick-up-point-delivery-order": "🚚",
};
```

That `Record<Union, T>` trick is the data counterpart to the assertNever
_control-flow_ trick — both make "you forgot a case" a compile error instead of
a runtime surprise. Both fit your "compiler as refactoring assistant" goal
exactly: you change one type, and the compiler walks you to every spot that now
needs attention.

## Regex: ECMAScript-flavor extras (backrefs, lookaround) — NOT in bash ERE

JS/TS `RegExp` is ECMAScript flavor (PCRE-ish): it HAS what POSIX ERE lacks — `\d \w \s`,
`(?:…)`, lookahead/lookbehind, backreferences. (bash `[[ =~ ]]` / `grep -E` are POSIX ERE: which have none.)

### Backreference IN the pattern: `\1` / `\k<name>`
```typescript
// \1 matches the SAME TEXT group 1 captured — not "the pattern again".
/(\w)\1/.test("hello");        // true  → "ll": \1 had to equal the captured "l"
/(\w)\1/.test("abc");          // false → no adjacent repeat
/(?<c>\w)\k<c>/.test("book");  // true  → "oo"; named backref is \k<name>
// Misread to avoid: /(\d)\1/ is NOT "two digits" — it's the SAME digit twice ("22", not "23").
```

### Backreference IN the replacement: `$1` / `$<name>` (different context!)
```typescript
"2024-01-15".replace(/(\d{4})-(\d{2})-(\d{2})/, "$3.$2.$1");  // "15.01.2024"
"2024-01".replace(/(?<y>\d{4})-(?<m>\d{2})/, "$<m>/$<y>");    // "01/2024"
// Gotcha: \1 in a replacement string is a LITERAL, not a backref. Pattern uses \1; replace uses $1.
```

### Lookahead: `(?=…)` positive, `(?!…)` negative — assert without consuming
```typescript
// Zero-width: checks what FOLLOWS but never becomes part of the match.
"foobar".match(/foo(?=bar)/)?.[0];  // "foo"  → matched foo; "bar" NOT in the result
"foobar".match(/foo(?!bar)/);       // null   → negative: fails because "bar" follows
"foobaz".match(/foo(?!bar)/)?.[0];  // "foo"  → passes: "bar" does not follow

/(?=.*\d)(?=.*[a-z]).{8,}/.test("abcdef12");  // → true
//  (?=.*\d)     from position 0, assert SOMEWHERE ahead there's a digit   (cursor doesn't move)
//  (?=.*[a-z])  from the SAME position 0, assert somewhere ahead a lowercase (rewinds, re-checks)
//  .{8,}        only NOW consume: ≥8 chars. Each (?=…) peeks & rewinds → it's "AND" in one pass.
```

1. **"Ahead"** is relative to the current position, not the string's start. At position 0
each `(?=.*…)` scans the whole string; if the engine were at position 3, they'd scan
from 3 onward. They rewind the cursor to where they started (zero-width), so the two
conditions are checked at the _same anchor_ point — that's what makes it an AND.

2. `.test()` stops at the first position where all three succeed — it doesn't sweep
every position. For `"abcdef12"` (exactly 8 chars) it resolves at position 0:
position 1 would leave only 7 characters, failing `.{8,}`, so the engine never even advances.
It's decided in one pass at the start.


### Lookbehind: `(?<=…)` positive, `(?<!…)` negative — same, backward
```typescript
"$100".match(/(?<=\$)\d+/)?.[0];    // "100" → the $ is required but NOT captured
"€50".match(/(?<!\$)\d+/)?.[0];     // "50"  → a number NOT preceded by $
// ES2018. Node/modern V8 (your bot's runtime) = fine; only ancient browsers lacked it.
```

### Pitfall: catastrophic backtracking (ReDoS)
```typescript
/(a+)+$/.test("aaaaaaaaaaaaaaaaaaaa!");
//  (a+)   matches a run of a's…
//  (…)+   …and the OUTER + lets it match MANY such runs, so "aaaa" splits as (aaaa),(aaa)(a),
//         (aa)(aa),(a)(a)(a)(a)… exponentially many ways.
//  $      the trailing "!" never matches end-of-string, so the engine BACKTRACKS through every
//         splitting before giving up → seconds+ on ~25 a's.
// Rule: never nest a quantifier inside another over the same class — (a+)+, (a*)*, (a|a)+.
//       Flatten to a+ / a*, or anchor to remove the ambiguity.
```

## Discriminated-union "Result" for parse-or-fail
Return a value OR an error as data — don't throw, don't do I/O (reply/log) inside the parser.
Keeps it pure-ish and unit-testable; the caller decides what to do with the error.
```typescript
type Parsed =
  | { ok: true; chatId: number; rest: string[] }
  | { ok: false; error: string };        // ready-to-send message, caller owns the reply

const parsed = parseCommandArgs(text, USAGE);
if (!parsed.ok) { await ctx.reply(parsed.error); return; } // .ok narrows the union
const { chatId, rest } = parsed;          // here TS knows the ok:true shape
```
> zod's `safeParse` returns this exact shape: `{ success: true, data } | { success: false, error }`.

## Type guard (`x is T`) vs assertion (`as T`)
A guard VERIFIES at runtime then narrows; `as` just tells the compiler to trust you (can lie).
```typescript
export function isNotificationSlug(s: string): s is NotificationType {
  return (VALID_SLUGS as readonly string[]).includes(s);   // real check
}
const [slug] = rest;                       // string | undefined
if (!slug || !isNotificationSlug(slug)) return;            // slug is NotificationType past here
const valid = rest.filter(isNotificationSlug);             // filter+guard: string[] -> NotificationType[]
```
> `rest as [NotificationType]` would compile but is false safety — runtime `rest` is still `string[]`.

## parseChatId — strict integer parse
```typescript
export function parseChatId(raw: string): number | null {
  const n = Number(raw);                   // strict: '123abc' -> NaN
  return Number.isInteger(n) ? n : null;   // rejects NaN AND floats ('12.9' -> null)
}
// parseInt is LENIENT: parseInt('123abc') -> 123, parseInt('12.9') -> 12. Wrong for a whole-token id.
// zod: const ChatId = z.coerce.number().int();  ChatId.safeParse(raw)
```
> Layering: keep `parseChatId` pure so commands that DON'T need auth (/addmanager) reuse it;
> `resolveManagerCommand` composes the manager-membership check on top.
