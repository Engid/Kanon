# Workshop: Error Handling

**Status:** Early design. Core decisions made, details to be refined during implementation.

---

## Approach

Clef uses **Result types** for recoverable errors and **panic** for unrecoverable errors. No exceptions, no try/catch.

## Result Type

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

## The `?` Operator

Borrowed from Rust. Unwraps `Ok` or early-returns `Err` from the enclosing function:

```
fun load_and_transpose(path: String, interval: Interval): Result[Music, String] {
    let raw = read_file(path)?            // early return if Err
    let scale = parse_scale(raw)?         // early return if Err
    let notes = scale.map(p => note(p, 1/4))
    Ok(notes.reduce((a, b) => a.seq(b)).transpose(interval))
}
```

## Unwrap (Panic on Error)

For cases where failure is unexpected or unrecoverable:

```
let scale = parse_scale(input).unwrap()          // panics with generic message
let scale = parse_scale(input).expect("bad input") // panics with custom message
```

## Panic

Unrecoverable errors halt execution with an error message. Sources of panic:

- Calling `.unwrap()` on an `Err` value
- Index out of bounds: `list(99)` on a 10-element list
- Failed exhaustive pattern match
- Explicit `panic("message")`
- Division by zero (integer): `1.div(0)`

Division of rationals by zero (`1/0`) is a compile-time or parse-time error since both operands are literals.

## Where Errors Typically Appear

Most Clef music code is pure computation — building Music trees, transforming them, chaining sequences. Errors are rare in this domain. The places where `Result` matters:

- File I/O (host-provided): `read_file`, `export`
- Parsing external input: scale definitions, MIDI files
- Host FFI calls that can fail
- User input validation in interactive/REPL contexts

## What's NOT Here (Yet)

- Error type hierarchy or standard error enum
- `try`/`catch` (intentionally omitted — use `match` on Results)
- Stack traces in panic messages (implementation detail)
- Zig-style `!` error union syntax (considered, but `Result[T, E]` with generics is clean enough)

## Open Questions

- Should `?` work on optional types (`T?`) in addition to `Result`? i.e., `let x = maybe_value?` returns `nil`/`None` early. Rust does this with `Option`, Swift does it with optional chaining. Probably yes.
- Should there be a `Result` shorthand? e.g., `fun foo(): Interval ! String` meaning `Result[Interval, String]`. Nice syntax but adds grammar complexity. Park for now.
