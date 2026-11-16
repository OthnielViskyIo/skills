---
paths:
  - "**/*.{ts,tsx}"
---

# TypeScript Rules

These rules govern all TypeScript code generated in this project. Follow them precisely and without exception unless the user explicitly overrides a specific rule.

---

## 1. Compiler Strictness

- Always assume `"strict": true` in `tsconfig.json`. This enables `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictPropertyInitialization`, and others simultaneously.
- Also enable `noUncheckedIndexedAccess`, `noImplicitReturns`, and `exactOptionalPropertyTypes` where the project allows it.
- Never emit code that produces type errors. If the project has `noEmitOnError: true`, treat all type errors as blocking.

---

## 2. Primitive Types

- Use lowercase `string`, `number`, `boolean`, `bigint`, `symbol` — never the wrapper types `String`, `Number`, `Boolean`.
- Use `number[]` or `Array<number>` interchangeably. Prefer `Type[]` shorthand for simple types; `Array<Type>` for complex or generic types.
- Use `readonly T[]` or `ReadonlyArray<T>` for arrays that should not be mutated.

---

## 3. Avoid `any`

- **Never use `any`** unless migrating legacy untyped JavaScript and there is no alternative.
- When a type is genuinely unknown, use `unknown` and narrow it before use.
- When accepting arbitrary functions without calling them, use `() => void`, not `Function`.
- When a value must be widened to pass through a boundary, prefer two-step assertion (`value as unknown as TargetType`) over a direct `as` cast between unrelated types.

```typescript
// ❌
function parse(input: any): any { /* ... */ }

// ✅
function parse(input: string): unknown { /* ... */ }
```

---

## 4. Type Inference

- Do not add type annotations where TypeScript can infer the type clearly (local variables initialized to a literal, arrow function return types in simple expressions).
- Always annotate: function parameters, return types of exported/public functions, class property declarations, and anything where inference would produce `any` or an overly broad type.

```typescript
// ❌ — redundant annotation
const name: string = "Alice";

// ✅ — annotation needed
function greet(name: string): string {
  return `Hello, ${name}`;
}
```

---

## 5. `interface` vs `type`

- Use `interface` for object shapes that represent a named concept, especially domain entities, value objects, and repository contracts. Interfaces are extendable and produce cleaner error messages.
- Use `type` for unions, intersections, mapped types, conditional types, and aliases for primitives.
- Never use `interface` to rename primitives.

```typescript
// ✅ interface for domain shape
interface Member {
  id: MemberId;
  email: string;
}

// ✅ type for union
type PaymentStatus = "pending" | "completed" | "failed";
```

---

## 6. Readonly and Immutability

- Mark interface/type properties `readonly` when they should not be reassigned after construction.
- Use `readonly` tuple types for fixed-length immutable data.
- Use `as const` to infer the narrowest literal type from object and array literals, especially for configuration objects and discriminant values.

```typescript
interface Circle {
  readonly kind: "circle";
  readonly radius: number;
}

const DIRECTIONS = ["north", "south", "east", "west"] as const;
type Direction = typeof DIRECTIONS[number];
```

---

## 7. Generics

Follow the three rules from the TypeScript handbook:

- **Push type parameters down**: prefer `T[]` over `T extends any[]` — the former lets TypeScript infer a concrete type rather than returning `any`.
- **Use fewer type parameters**: do not introduce a type parameter solely to name a value that doesn't relate to another type.
- **Type parameters should appear twice**: if a type parameter appears only once in the signature, the function probably doesn't need to be generic.

```typescript
// ❌ — Func adds no information
function filter<Type, Func extends (arg: Type) => boolean>(
  arr: Type[],
  func: Func
): Type[] { /* ... */ }

// ✅
function filter<Type>(arr: Type[], func: (arg: Type) => boolean): Type[] { /* ... */ }
```

- Constrain type parameters with `extends` when the function requires specific properties.
- Provide generic parameter defaults (`<T = DefaultType>`) when most callers use the same type.

---

## 8. Union Types and Discriminated Unions

- Model variants (e.g. different subtypes, states, or commands) as discriminated unions with a literal `kind` (or `type`) property. This enables exhaustive narrowing.

```typescript
interface Circle { kind: "circle"; radius: number; }
interface Square { kind: "square"; sideLength: number; }
type Shape = Circle | Square;
```

- Add an exhaustiveness check in `switch` statements over discriminated unions using `never`:

```typescript
function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle": return Math.PI * shape.radius ** 2;
    case "square": return shape.sideLength ** 2;
    default:
      const _exhaustive: never = shape;
      return _exhaustive;
  }
}
```

- Prefer union types over function overloads when the parameter can be expressed as a union and the return type does not change.

```typescript
// ❌ overloads when a union suffices
function len(s: string): number; function len(arr: readonly unknown[]): number;
// ✅
function len(x: string | readonly unknown[]): number { return x.length; }
```

---

## 9. Narrowing

- Use `typeof` guards for primitives, `instanceof` for class instances, `in` for structural presence.
- Use custom type predicates (`pet is Fish`) when reusable narrowing logic is extracted to a function.
- Avoid truthiness narrowing on strings (`if (str)`) — it silently skips empty strings. Prefer `str !== undefined && str !== null` or `str != null`.
- Rely on TypeScript's control flow analysis: after a check branches, use the narrowed type rather than re-asserting.

---

## 10. Null and Undefined

- With `strictNullChecks` on, always handle `null` and `undefined` explicitly — do not use the non-null assertion operator (`!`) except where it is provably safe and the reason is commented.
- Prefer optional chaining (`?.`) and nullish coalescing (`??`) over manual `!` assertions.
- Use `value != null` (loose equality) to guard against both `null` and `undefined` simultaneously.

---

## 11. Functions

- Always annotate function parameter types. Return types should be annotated on public/exported functions.
- Use `void` as the return type for functions with no meaningful return value.
- Do not mark callback parameters optional unless the implementation genuinely omits them — optional callback params cause type errors at the call site.
- Use `unknown` (not `any`) for function parameters that accept arbitrary input and require internal narrowing.
- Use `never` as the return type for functions that always throw or loop forever.

---

## 12. Type Assertions

- Minimise `as` type assertions. They suppress the type checker and can mask bugs.
- When an assertion is necessary (e.g. after a DOM query), use the narrowest possible type and add a comment explaining why it is safe.
- Never use `as any` as a shortcut — widen through `as unknown as TargetType` and justify it.

---

## 13. Object Types and Index Signatures

- When an object's keys are dynamic, type the index signature value as `unknown` rather than `any`: `{ [key: string]: unknown }`.
- Avoid excess property workarounds (casting with `as` or assigning to an intermediate variable) — treat excess property errors as real bugs.
- Use `Record<Keys, ValueType>` as a shorthand for index signatures over a known key union.

---

## 14. Branded Types for Domain Primitives

- Never type a domain identifier or constrained value as bare `string`/`number`. Brand it so unrelated ids cannot be swapped.

```typescript
declare const brand: unique symbol;
type Brand<T, B> = T & { readonly [brand]: B };

export type MemberId = Brand<string, "MemberId">;

export function memberId(value: string): MemberId {
  if (value.length === 0) throw new Error("MemberId must not be empty");
  return value as MemberId; // safe: validated above, brand is phantom
}
```

---

## 15. Enums

- Prefer string literal union types over TypeScript `enum`. Enums emit runtime code and can behave unexpectedly (numeric enums, ambient const enums across modules).
- Use `const enum` only when the consuming environment guarantees the enum is inlined.

```typescript
// ❌
enum Direction { North, South }

// ✅
type Direction = "north" | "south";
```

---

## 16. Classes

- Use `readonly` on class properties that are set only in the constructor.
- Prefer `private` declarations for encapsulation; use `#privateField` syntax when true runtime privacy is needed.
- Avoid `public` as an explicit modifier — it is the default and adds noise.
- Do not use abstract classes as a substitute for interfaces; reserve them for template method patterns where a concrete partial implementation exists.

---

## 17. Module Boundaries and Exports

- Export only what needs to be public. Prefer named exports over default exports for better refactoring support.
- Prefer `import type { Foo }` for type-only imports — this improves tree-shaking and is required in some bundler configurations.
- Do not re-export everything from a barrel file (`export * from`) without intent — it increases coupling and makes tree-shaking harder.

---

## 18. Comments and Documentation

- Use JSDoc (`/** */`) on all exported types, interfaces, and functions. Describe intent, not implementation.
- Do not use `@ts-ignore`; use `@ts-expect-error` with a comment explaining the suppression.
- If a workaround is needed due to a TypeScript limitation, link to the relevant issue or explain the constraint.
