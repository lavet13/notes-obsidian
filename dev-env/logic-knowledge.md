---
id: logic-knowledge
aliases: []
tags: []
---

# Logic Knowledge

## XOR vs inclusive OR
- **OR (inclusive, ∨)** — at least one true. **Both is allowed.** `T∨T = T`
- **XOR (exclusive, ⊕)** — exactly one true. Both or neither is false. `T⊕T = F`
> Mnemonic: XOR = "one or the other, **but not both**".
> Tell in code: `if (!a && !b) error` rejects only NEITHER → that's inclusive OR
> ("at least one required"), NOT XOR. XOR must also reject BOTH.

## Two-flag cheat sheet (JS)
```javascript
!!a || !!b      // at least one   (OR)
!!a !== !!b     // exactly one    (XOR) — coerce, then "they differ"
!(a && b)       // at most one    (NAND) — "optional but mutually exclusive"
!!a && !!b      // both           (AND)
!a && !b        // neither        (NOR)
```

## N operands: count, don't chain
```javascript
!!a !== !!b !== !!c            // ✗ PARITY (true when an ODD number are true) — T,T,T passes!
const n = [a, b, c].filter(Boolean).length;
n === 1   // exactly one (real N-ary XOR)
n >= 1    // at least one
n <= 1    // at most one
n === 0   // none
```
> `!!a !== !!b` works for two only because parity and "exactly one" coincide at N=2.
