# Code Review Checklist – TypeScript

- [ ] All variables, function parameters, and return types are explicitly typed.
- [ ] No usage of `any` unless strictly necessary and justified.
- [ ] All types and interfaces are idiomatic (use type aliases for unions/intersections, interface for structures).
- [ ] Narrowing and type guards used where needed to ensure safety.
- [ ] No excessive type assertions or `as` casts.
- [ ] Functions and methods are typed at the boundary (public API/exports).
- [ ] Prefer enums or union literal types for discriminated values.
- [ ] No unnecessary use of `non-null assertion` (!).
- [ ] Correct use of generics—no overcomplicated/invented types.
- [ ] Leverage utility types (Partial, Record, Pick, etc.) where appropriate.
- [ ] No dead/redundant types left in the codebase.
- [ ] Ensure all third-party imports have corresponding type definitions.
- [ ] `tsconfig.json` settings are enforced and no `@ts-ignore` without justification.
- [ ] Strict mode (`"strict": true`) is enabled in `tsconfig.json`.
