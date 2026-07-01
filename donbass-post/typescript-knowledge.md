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
