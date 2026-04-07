# RFC-0002: Error Handling (`Result` Types and `panic`)

**Status:** Draft  
**Created:** 2026-04-05  
**Related:** §4 Expressions, §5 Standard Library

---

## Summary

Kanon uses **Result types** for recoverable errors and **panic** for unrecoverable errors. No exceptions, no try/catch. The `?` operator provides ergonomic early-return on errors.

---

## Motivation

Most Kanon music code is pure computation — building Music trees, transforming them, chaining sequences. Errors are rare in this domain. But at system boundaries (file I/O, parsing external input, host FFI), failure is possible and must be handled explicitly.

Exception-based error handling (try/catch) conflates control flow with error signaling, makes error paths invisible in function signatures, and is difficult to reason about in functional-style code. Result types make errors explicit in the type system — a function that can fail says so in its return type.

---

## Design

### Result Type

```
enum Result[T, E] {
    Ok(value: T),
    Err(error: E),
}
```

Functions that can fail return `Result`:

```
fun parse_scale(input: String): Result[List[Interval], String] {
    // ...
}
```

### The `?` Operator

Borrowed from Rust. Unwraps `Ok` or early-returns `Err` from the enclosing function:

```
fun load_and_transpose(path: String, interval: Interval): Result[Music, String] {
    let raw = read_file(path)?            // early return if Err
    let scale = parse_scale(raw)?         // early return if Err
    let notes = scale.map(p => note(p, 1/4))
    Ok(notes.reduce((a, b) => a.seq(b)).transpose(interval))
}
```

### Unwrap (Panic on Error)

For cases where failure is unexpected or unrecoverable:

```
let scale = parse_scale(input).unwrap()          // panics with generic message
let scale = parse_scale(input).expect("bad input") // panics with custom message
```

### Panic

Unrecoverable errors halt execution with an error message. Sources of panic:

- Calling `.unwrap()` on an `Err` value
- Index out of bounds: `list(99)` on a 10-element list
- Failed exhaustive pattern match
- Explicit `panic("message")`
- Division by zero (integer): `1.div(0)`

Division of rationals by zero (`1/0`) is a compile-time or parse-time error since both operands are literals.

---

## Where Errors Typically Appear

| Domain | Examples |
|--------|----------|
| File I/O (host-provided) | `read_file`, `export` |
| Parsing external input | Scale definitions, MIDI files |
| Host FFI calls | Any host function that can fail |
| User input validation | Interactive/REPL contexts |

---

## Prior Art

| Language | Mechanism | Notes |
|----------|-----------|-------|
| **Rust** | `Result<T, E>` + `?` | Direct inspiration. Proven ergonomic. |
| **Go** | Multiple return values | Verbose, no type-level enforcement |
| **Haskell** | `Either a b` + monadic do | Powerful but complex for beginners |
| **Swift** | `throws` + `try`/`catch` | Exception-style syntax over value types |
| **Zig** | `!T` error unions | Compact syntax, similar semantics to Result |

Kanon follows Rust most closely — `Result[T, E]` with generic type parameters and the `?` operator for ergonomic propagation.

---

## What's NOT Here (Yet)

- Error type hierarchy or standard error enum
- `try`/`catch` (intentionally omitted — use `match` on Results)
- Stack traces in panic messages (implementation detail)
- Zig-style `!` error union syntax (considered, but `Result[T, E]` with generics is clean enough)

---

## Open Questions

1. **Should `?` work on optional types (`T?`) in addition to `Result`?** i.e., `let x = maybe_value?` returns `nil`/`None` early. Rust does this with `Option`, Swift does it with optional chaining. Probably yes.

2. **Should there be a `Result` shorthand?** e.g., `fun foo(): Interval ! String` meaning `Result[Interval, String]`. Nice syntax but adds grammar complexity. Park for now.

---

## Decisions Requested

- [ ] **`?` on optionals** — Should `?` work on `T?` / `Option[T]` in addition to `Result`?
- [ ] **Result shorthand** — Add `T ! E` syntax as sugar for `Result[T, E]`?
- [ ] **Standard error type** — Define a builtin `Error` enum or leave error types user-defined?
- [ ] **v0 priority** — Ship in v0, or defer?
