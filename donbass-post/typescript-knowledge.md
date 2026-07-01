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

---

# Anki's cards

Card 1 - assertNever (recall)

Front: TypeScript - what is the assertNever pattern and how does it enforce exhaustiveness?
Back: function assertNever(x: never): never { throw new Error(...) }. In a switch, handling-then-returning each case narrows the union; by default it is never. Passing the value to assertNever in the default only compiles if TS believes it is never - so adding an unhandled union member makes the default a COMPILE error.

Card 2 - assertNever (application)

Front: TypeScript - you add a new status token to a union that a switch statement handles. How do you guarantee the compiler forces you to handle it?
Back: Put default: return assertNever(result) in the switch. The new member is no longer narrowed away, so result is not never at the default, and assertNever(x: never) fails to compile - pointing you exactly at the switch that needs the new case.

Card 3 - Record<Union,T> (recall)

Front: TypeScript - what is the data-side counterpart to the assertNever control-flow trick?
Back: Record<Union, T> - an object keyed by the union forces an entry for EVERY member. Add a union member and the object will not compile until you add its entry. (Example: an ICONS map keyed by notification-type union.)

Card 4 - compiler as assistant (application)

Front: TypeScript - you want "you forgot a case" to be a compile error instead of a runtime surprise, both for control flow and for data. Which two tools?
Back: Control flow: assertNever(x: never) in a switch default. Data: Record<Union, T> for a lookup object. Both make the compiler walk you to every spot that needs attention when you change one type.
