
`as T` and `!` tell the compiler “trust me.”

Prefer:

* Runtime checks (`typeof`, `instanceof`, `in`).
* Explicit null handling.

Assertions are acceptable at trust boundaries with a clear reason.

Always use `as T`, not angle-bracket syntax.

---

## 4. Let Type Inference Work

Avoid redundant annotations:

```ts
const count = 0;   // not: const count: number = 0;
```

Add explicit types when:

* The type cannot be inferred (e.g., empty arrays).
* Exported/public APIs.
* Complex return types.
* You want to catch object literal typos.

Always annotate exported functions’ return types.

---

## 5. Use `satisfies` for Config Objects

`satisfies` validates shape without widening literal types.

Use when:

* You want validation.
* You want to preserve literal inference.

Use `: Type` when the variable must be treated as the wider type.

---

## 6. Use `as const` for Immutable Literals

`as const`:

* Preserves literal types.
* Makes structures deeply readonly.

Useful for:

* Config objects
* Lookup tables
* Enum-like patterns

Combine with `satisfies` for validated immutable configs.

---

## 7. Handle `null` and `undefined` Explicitly

Prefer:

```ts
interface Config {
  timeout?: number;
}
```

Over:

```ts
timeout: number | undefined;
```

Rules:

* Narrow before use; never silence null with `!`.
* Use `??` instead of `||` for defaults.
* Avoid hiding nullability inside type aliases.
* Enable `noUncheckedIndexedAccess`.

---

## 8. `interface` vs `type`

Prefer:

* `interface` for object shapes.
* `type` for unions, intersections, mapped, conditional, tuple, and function types.

Be consistent.

Remember: interfaces support declaration merging.

---

## 9. Use Utility Types Wisely

Core utilities:

* `Partial<T>`
* `Required<T>`
* `Pick<T, K>`
* `Omit<T, K>`
* `Record<K, V>`
* `Readonly<T>`
* `ReturnType<T>`
* `Parameters<T>`
* `Awaited<T>`
* `NonNullable<T>`
* `Extract<T, U>`
* `Exclude<T, U>`

Use utilities when the derived type must stay in sync.

Avoid deeply nested, unreadable combinations.

---

## 10. Use Generics Carefully

A good generic:

* Appears in multiple positions (input + output).
* Is constrained with `extends`.
* Has a clear meaning.

Avoid:

* Return-type-only generics.
* Unconstrained `T`.
* Generics that don’t affect behavior.

If you can’t explain what `T` represents clearly, simplify.

---

## 11. Prefer Discriminated Unions Over Boolean Flags

Instead of:

```ts
{ isLoading: boolean; isError: boolean }
```

Use:

```ts
type State =
  | { status: 'loading' }
  | { status: 'success'; data: Data }
  | { status: 'error'; error: Error };
```

Always exhaustively handle with `switch`. Use `never` for compile-time enforcement.

---

## 12. Use Template Literal Types When Justified

Good for:

* Event naming conventions
* Route parameters
* Structured string patterns (e.g., CSS units)

Avoid if a simple string union is sufficient.

---

## 13. Master Type Narrowing

Use:

* `typeof`
* `instanceof`
* `in`
* Truthiness checks
* Equality checks

Write user-defined type guards:

```ts
function isUser(x: unknown): x is User { ... }
```

Use assertion functions (`asserts`) when appropriate.

---

## 14. Variable Declarations

* Use `const` by default.
* Use `let` only when reassigning.
* Never use `var`.
* One declaration per line.
* Avoid deep destructuring.
* Prefer shallow destructuring + clarity.

---

## 15. Functions

* Use function declarations for top-level functions.
* Use arrow functions for callbacks.
* Avoid passing functions like `parseInt` directly to array methods.
* Prefer default parameters over optional + manual fallback.
* Use overloads only when return type depends on input type.

---

## 16. Async/Await

Prefer `async/await` over promise chains.

Rules:

* Never leave floating promises.
* Use `await`, `.catch()`, or `void`.
* Use `Promise.all` for independent concurrency.
* Use `Promise.allSettled` for partial failure.
* Always annotate exported async return types.
* Don’t mark a function `async` if nothing is awaited.

---

## 17. Classes

* Use `readonly` for immutable fields.
* Use parameter properties to reduce boilerplate.
* Don’t write `public` unless necessary.
* Use `override` when overriding.
* Prefer composition over inheritance.
* Use `#private` only when runtime privacy is required (e.g., libraries).

---

## 18. Enums

Prefer string unions:

```ts
type Status = 'pending' | 'active';
```

Use enums only when you need a runtime object.

Rules:

* Prefer string enums.
* Avoid numeric enums.
* Avoid `const enum`.

You can emulate enums with `as const` objects.

---

## 19. Imports & Exports

* Prefer named exports.
* Avoid default exports.
* Use `import type` for type-only imports.
* Keep import order consistent.
* Use barrel files only at module boundaries.
* Avoid circular dependencies.

---

## 20. Naming Conventions

* `UpperCamelCase`: types, classes, enums.
* `lowerCamelCase`: variables, functions.
* `UPPER_SNAKE_CASE`: deeply immutable constants only.
* No Hungarian notation.
* No `I` prefix for interfaces.
* No `_private` naming — use access modifiers.
* Boolean names should read like predicates (`isActive`, `hasAccess`).

---

## 21. Error Handling

* Always throw `Error` objects.
* Create domain-specific error classes.
* Catch errors as `unknown`.
* Narrow before using.
* Never leave empty catch blocks without explanation.
* Keep `try` blocks small.
* Consider `Result<T, E>` pattern when failures are expected.

---

## 22. Comments & Documentation

* Use TSDoc for public APIs.
* Comment **why**, not **what**.
* Use `TODO`, `FIXME`, `HACK` consistently.
* Never leave commented-out code.

---

## 23. Testing

* Type test fixtures.
* Use factory functions with `Partial<T>` overrides.
* Avoid `as any`.
* Test behavior, not types (unless explicitly testing types).
* Use type-testing tools when needed.

---

## 24. Module Organization

* One responsibility per file.
* Organize by feature.
* Use barrel files only for public API boundaries.
* Avoid circular imports.

---

## 25. Simplicity Is the Primary Rule

Overly complex types reduce maintainability.

Avoid:

* Deeply nested types.
* Overly complex unions/intersections.
* Unnamed conditional/mapped types.
* Deep generic chains.

Prefer:

* Named intermediate types.
* Explicit over clever.
* Types you can explain in one sentence.

If a teammate cannot quickly understand a type, simplify it.

---

## Core Principles Summary

1. Enable strict mode.
2. Eliminate `any`.
3. Avoid unnecessary assertions.
4. Let inference work.
5. Model state explicitly with discriminated unions.
6. Handle nulls intentionally.
7. Avoid floating promises.
8. Prefer string unions over enums.
9. Use named exports.
10. Keep types simple and readable.

TypeScript is a tool for reducing bugs and increasing clarity. If your types make the code harder to understand, you are using it incorrectly.
