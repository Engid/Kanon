# RFC-0001: Bounded Integer Types (`Wrap` and `Clamp`)

**Status:** Draft  
**Created:** 2026-04-06  
**Related:** workshop-numeric-literals.md, §3.1 Numeric Tower, §3.2 Music Types

---

## Summary

Add two parameterized integer types to Kanon:

- `Wrap[N]` — a modular integer in the range `0..N-1`. Arithmetic wraps via `mod N`.
- `Clamp[N]` — a bounded integer in the range `0..N-1`. Arithmetic saturates at the bounds.

`N` must be a compile-time constant. `Wrap[12]` and `Wrap[19]` are distinct types.

---

## Motivation

### EDO pitch classes

An EDO step is fundamentally a modular integer. In 12-EDO, step 7 + step 7 = step 2 (octave equivalence). The current design represents Edo as a product type `{ step: Int, divisions: Int }`, but this doesn't capture the modular arithmetic — you'd need to manually `% divisions` after every operation. `Wrap[12]` makes modular arithmetic automatic and type-safe.

```
type Edo12 = Wrap[12]

let fifth: Edo12 = 7
fifth + fifth        // → 2  (14 mod 12)
fifth + Wrap[19](7)  // ERROR — different types
```

### Safe array indexing

A `Wrap[N]` value can never be out of bounds for an array of size `N`. The compiler can elide bounds checks when indexing with a matching `Wrap` type.

```
let buffer: Array[Float, 8] = ...
let i: Wrap[8] = ...
buffer[i]            // always safe — no runtime bounds check needed
```

### Everything else this unlocks

| Use case | Type | Behavior |
|----------|------|----------|
| 12-EDO pitch classes | `Wrap[12]` | Octave equivalence via mod |
| Ring buffer pointers | `Wrap[N]` | Auto-wrapping write head |
| Clock arithmetic | `Wrap[24]`, `Wrap[60]` | Hours, minutes |
| MIDI note numbers | `Clamp[128]` | 0–127, saturates |
| MIDI channels | `Clamp[16]` | 0–15, saturates |
| 8-bit color channels | `Clamp[256]` | 0–255, saturates |
| Toroidal game grids | `Wrap[W]`, `Wrap[H]` | Wrapping coordinates |

These are general-purpose types that happen to solve the EDO representation problem perfectly.

---

## Prior Art

### Cmajor: `wrap<N>` and `clamp<N>`

Cmajor has exactly this feature. `wrap<N>` and `clamp<N>` are built-in integer types where `N` is a compile-time constant. Primary motivation: safe array indexing without runtime bounds checks.

```c
wrap<5> w = 0;
clamp<5> c = 0;

loop (7) { ++w; ++c; }
// w == 2  (wrapped)
// c == 4  (clamped)

w = 4 - 5;  // w == 4  (mod)
c = 4 - 5;  // c == 0  (clamped to min)
```

Cmajor also uses `wrap<N>` for `for` loop iteration:
```c
for (wrap<5> i)
    console <- i;  // prints 0, 1, 2, 3, 4
```

In Cmajor, `wrap` and `clamp` are **built-in primitive types**, not library-defined. `N` must be a compile-time constant (Cmajor uses angle brackets `<>` for type parameters).

### Ada: Range types and modular types

Ada has both concepts as first-class language features:

```ada
type Byte is mod 256;              -- modular (wrapping) integer
type MIDI_Note is range 0 .. 127;  -- range (clamping) integer, raises Constraint_Error on overflow
```

Ada's modular types are unsigned and wrap on overflow. Range types raise an exception on overflow rather than clamping, but the concept is similar.

### Rust: No built-in bounded integers

Rust has `wrapping_add()`, `saturating_add()` etc. as methods on integer types, but no type-level bounded integers. The `modular` crate exists but it's a library solution without compiler support for optimized indexing. Rust's const generics (`[T; N]`) provide the foundation — a `Wrap<N>` struct could be built in userland.

### Pascal / Modula-2: Subrange types

```pascal
type Hour = 0..23;
type Dice = 1..6;
```

Subrange types with compile-time bounds. Overflow is a runtime error, not wrap or clamp. No modular arithmetic.

### Zig: Comptime integers

Zig supports arbitrary integer sizes via `u4`, `i12`, etc. — these are fixed-width with wrapping/overflow behavior controllable via operators (`+%` for wrapping add, `+|` for saturating add). Not directly parameterized by a bound, but the `@mod` builtin and explicit bit widths serve a similar role.

---

## Design Exploration

### The core question: Builtin or library?

Can `Wrap[N]` be defined in Kanon's stdlib using existing language features, or does it require compiler support?

#### What `Wrap[N]` needs:

1. **A compile-time integer parameter** — `N` must be part of the type, known at compile time, so that `Wrap[12]` ≠ `Wrap[19]`.
2. **Custom arithmetic** — `+`, `-`, `*` apply `mod N`. This is not standard dimensional arithmetic (like `unit` provides).
3. **Conversion from Int** — `Wrap[12](15)` → `3`.
4. **Type-safe mixing prevention** — `Wrap[12] + Wrap[19]` is a compile error.

#### What Kanon currently has:

- **Generic type parameters** — `[T]`, `[T, U]` — but only type identifiers, NOT compile-time integer values.
- **`op_` convention** (proposed) — user-defined operator overloading via named functions.
- **`unit` keyword** (proposed) — branded numeric types with auto-derived arithmetic, but the arithmetic rules are dimensional (same-unit add, scale by dimensionless), not modular.

#### Gap: Kanon has no const generics

The current generic system only supports type parameters:

```
type Box[T] { value: T }          // ✅ T is a type
fun first[T](seq: Seq[T]): T     // ✅ T is a type
```

There's no way to write:

```
type Wrap[N: Int] { value: Int }  // ❌ N is a value, not a type
```

Without this, `Wrap` cannot be a library type. It must either be:

1. **A builtin type** (like Cmajor) — the compiler knows about `Wrap` and `Clamp` specially.
2. **Enabled by adding const generics** to the language, then defined in stdlib.

### Option A: Builtin types (Cmajor approach)

`Wrap[N]` and `Clamp[N]` are primitive types known to the compiler, alongside `Int`, `Float`, `Ratio`, `Bool`, `String`.

```
let x: Wrap[12] = 7
let y: Clamp[128] = 100
```

The compiler hardcodes their arithmetic semantics. No user-defined types can replicate this behavior.

**Pros:**
- Simplest to implement — no new type system features needed
- Compiler can optimize: elide bounds checks when indexing arrays with matching `Wrap` types
- Clear, predictable behavior

**Cons:**
- Not user-extensible — you can't define your own types with similar compile-time parameterization
- Adds two more primitive types to the language
- Doesn't solve the broader problem (fixed-size arrays, compile-time matrix dimensions, etc.)

### Option B: Add const generics, then define in stdlib

Add a `const` modifier to generic type parameters, allowing compile-time values:

```
type Wrap[const N: Int] {
    value: Int
}

fun op_add(a: Wrap[N], b: Wrap[N]): Wrap[N] =
    Wrap[N] { value: (a.value + b.value) % N }

fun op_sub(a: Wrap[N], b: Wrap[N]): Wrap[N] =
    Wrap[N] { value: ((a.value - b.value) % N + N) % N }

// etc.
```

This requires the compiler to:
- Accept `const IDENT: Type` in type parameter lists
- Monomorphize: `Wrap[12]` and `Wrap[19]` become distinct concrete types
- Allow `N` to appear in expressions inside the type body and associated functions

**Pros:**
- `Wrap` and `Clamp` become stdlib, not builtins — growable language
- Unlocks fixed-size arrays: `type Array[T, const N: Int]`
- Unlocks compile-time-parameterized types generally: `Matrix[Float, 4, 4]`
- Follows Rust's const generics model (well-understood, proven)
- Aligns with Steele's growability principle

**Cons:**
- Significantly more complex to implement — const generic monomorphization is nontrivial
- Needs compile-time evaluation of expressions involving const params (`(a + b) % N`)
- May be overkill for v0 if Wrap/Clamp are the only use case
- Rust took years to stabilize const generics — there are real design subtleties

### Option C: Hybrid — builtin for v0, const generics later

Ship `Wrap[N]` and `Clamp[N]` as builtins in v0. When/if const generics are added later, migrate them to stdlib definitions. The user-facing syntax and semantics stay identical either way.

This is pragmatic: get the feature now, pay the design cost later. The risk is that builtin behavior might diverge from what const generics would allow, but if the surface area is small (just arithmetic + comparison + conversion), the migration should be clean.

**Pros:**
- Fast to implement
- Doesn't block on a major type system feature
- Migration path is clear

**Cons:**
- Technical debt: builtin behavior must later match what stdlib + const generics would produce
- Two implementations of the same thing over the language's lifetime

### Option D: Type-level naturals via phantom types (no const generics)

Encode the bound as a type rather than a value, using phantom types:

```
type N12 {}
type N19 {}
type N128 {}

type Wrap[N] { value: Int }
```

`Wrap[N12]` and `Wrap[N19]` are distinct types because `N12 ≠ N19`. But the actual modulo value isn't carried by the type — you'd need a typeclass/trait to associate `N12` with the integer `12`:

```
trait BoundValue[N] {
    fun bound(): Int
}

impl BoundValue[N12] { fun bound(): Int = 12 }
impl BoundValue[N19] { fun bound(): Int = 19 }
```

**Pros:**
- Works with existing type system (no const generics needed)
- Type safety: `Wrap[N12] + Wrap[N19]` is a type error

**Cons:**
- Extremely verbose — user must define a phantom type AND a trait impl for every bound
- `Wrap[12]` syntax doesn't work — it would be `Wrap[N12]`
- No way to write generic functions over "any Wrap" without higher-kinded types
- Feels like a workaround, not a real solution

**Verdict:** Not worth it. This is the kind of encoding that signals a missing language feature.

---

## Proposed Approach

**Option C (hybrid)** for v0, with a clear eye toward Option B.

### v0: Builtin `Wrap[N]` and `Clamp[N]`

The compiler recognizes `Wrap` and `Clamp` as builtin parameterized types. `N` must be a positive integer literal or a previously-defined compile-time constant.

#### Type rules

```
Wrap[N]  where N > 0     // modular integer, range 0..N-1
Clamp[N] where N > 0     // saturating integer, range 0..N-1
```

`Wrap[12]` ≠ `Wrap[19]` ≠ `Clamp[12]`. All three are distinct types.

#### Construction

```
let a: Wrap[12] = 7          // literal in range — OK
let b: Wrap[12] = 15         // out of range — wraps to 3
let c = Wrap[12](15)         // explicit construction — wraps to 3

let d: Clamp[128] = 200      // out of range — clamps to 127
let e = Clamp[128](-5)       // clamps to 0
```

#### Arithmetic

| Expression | `Wrap[N]` result | `Clamp[N]` result |
|-----------|-----------------|-------------------|
| `a + b` | `(a + b) mod N` | `min(a + b, N-1)` |
| `a - b` | `((a - b) mod N + N) mod N` | `max(a - b, 0)` |
| `a * b` | `(a * b) mod N` | `min(a * b, N-1)` |
| `-a` | `(N - a) mod N` | `0` (clamp at min) |
| `a == b` | standard | standard |
| `a < b` | standard | standard |

Mixed operations:

| Expression | Result |
|-----------|--------|
| `Wrap[12] + Wrap[12]` | `Wrap[12]` |
| `Wrap[12] + Wrap[19]` | **ERROR** — different types |
| `Wrap[12] + Int` | `Wrap[12]` (Int is wrapped into range first) |
| `Int + Wrap[12]` | `Wrap[12]` |
| `Wrap[12] + Float` | **ERROR** — Wrap is integer-only |

#### Conversion

```
Int(myWrap)              // unwrap to Int — always safe
Wrap[12](myInt)          // wrap Int into range
Wrap[12](myClamp)        // ERROR — explicit: Int(myClamp) first, then Wrap[12](...)

let n: Wrap[12] = 7
n.bound                  // → 12 (the N parameter, as an Int)
```

#### Array indexing integration

When an array has a known size `N`, indexing with `Wrap[N]` or `Clamp[N]` is always safe:

```
let notes: list[String] = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"]
let step: Wrap[12] = 7
notes[step]              // "G" — compiler knows this is in bounds
```

This is aspirational for v0 — the compiler can emit the bounds-check-elision optimization later. But the type relationship should be established from the start.

### Future: Migrate to const generics (Option B)

When Kanon adds const generic parameters, `Wrap` and `Clamp` move from builtins to stdlib:

```
// In stdlib — hypothetical future
type Wrap[const N: Int] {
    value: Int
}

fun Wrap[const N: Int](raw: Int): Wrap[N] =
    Wrap[N] { value: ((raw % N) + N) % N }

fun op_add[const N: Int](a: Wrap[N], b: Wrap[N]): Wrap[N] =
    Wrap[N]((a.value + b.value) % N)

// ... etc
```

User-facing syntax and semantics remain identical. The migration is internal.

---

## EDO Revisited

With `Wrap[N]`, the music stdlib simplifies:

```
// Before (product type approach):
type Edo = { step: Int, divisions: Int }
fun edo(step: Int, divisions: Int): Edo = Edo { step, divisions }

// After (Wrap approach):
type Edo12 = Wrap[12]
type Edo19 = Wrap[19]
type Edo31 = Wrap[31]
```

No custom type needed. An EDO step IS a wrapping integer. The type captures both the value and the bound.

For generic EDO operations, you'd write functions that accept any `Wrap[N]`:

```
// Convert an EDO step to a frequency ratio (equal temperament)
fun to_ratio[const N: Int](step: Wrap[N]): Float =
    2.0 ** (Float(Int(step)) / Float(N))

// Interval between two steps
fun interval[const N: Int](a: Wrap[N], b: Wrap[N]): Wrap[N] =
    b - a

// UFCS usage
let fifth: Wrap[12] = 7
fifth.to_ratio()        // → 1.4983...  (2^(7/12))
```

The `7.edo(12)` UFCS syntax from the workshop doc could still exist as sugar that constructs a `Wrap[12]`:

```
fun edo(step: Int, divisions: Int): ???
// Problem: return type depends on runtime value `divisions`
// This can't return Wrap[divisions] without dependent types
```

This is a real tension. `7.edo(12)` requires the `12` to be a comptime value if the return type is `Wrap[12]`. Options:

1. **`edo` is a macro or comptime function** — `7.edo(12)` where `12` must be a literal → returns `Wrap[12]`
2. **Monomorphized overloads** — `fun edo12(step: Int): Wrap[12]`, `fun edo19(step: Int): Wrap[19]` — verbose
3. **Accept the limitation** — use explicit types: `let step: Wrap[12] = 7` or `Wrap[12](7)`
4. **Runtime fallback** — `edo(7, 12)` returns a runtime-checked bounded int (loses type safety)

**Recommendation:** Option 3 is cleanest. The explicit type annotation `Wrap[12](7)` or `let step: Wrap[12] = 7` is clear enough. A convenience alias in the music stdlib covers the common cases:

```
type Edo12 = Wrap[12]
type Edo19 = Wrap[19]
type Edo31 = Wrap[31]

let fifth: Edo12 = 7
```

---

## Open Questions

1. **Should `Wrap[N]` support non-zero lower bounds?** E.g., `Wrap[1, 13]` for range 1..12. Cmajor doesn't — always 0..N-1. Ada range types do. Keeping it 0-based is simpler and covers the primary use cases.

2. **What happens with `Wrap[0]` or `Wrap[-3]`?** Compile error. `N` must be a positive integer.

3. **Should `Clamp[N]` raise a warning on overflow?** Or silently saturate? Silent saturation is the point of the type — if you wanted errors, use a range check.

4. **Division and modulo on Wrap types?** `Wrap[12](7) / Wrap[12](3)` → ? Modular division requires modular inverse, which only exists when `N` is prime. Probably: don't auto-derive `/` and `%` for Wrap. Users can define `op_div` if they need it.

5. **How does this interact with `unit`?** Could you write `unit Edo12 = Wrap[12]`? Probably not useful — `Wrap[12]` already has the right arithmetic. `unit` adds dimensional branding to types with *standard* arithmetic. `Wrap` already has *custom* arithmetic.

6. **Literal syntax?** Should `7w12` or `7_12` exist as sugar for `Wrap[12](7)`? Probably not — it's not common enough to warrant literal syntax, and the explicit form is clear.

7. **Relation to fixed-size arrays?** If const generics are added later, `Array[T, const N: Int]` and `Wrap[N]` share the same mechanism. `Wrap[N]` is always a valid index for `Array[T, N]`. This is a strong argument for eventually moving to const generics.

---

## Decisions Requested

- [ ] **Builtin vs. const generics** — Option A, B, or C?
- [ ] **0-based only** — Always `0..N-1`, or support arbitrary ranges?
- [ ] **Division** — Auto-derive `/` and `%` for Wrap/Clamp, or leave them out?
- [ ] **Naming** — `Wrap`/`Clamp`, or something else? (`Mod`/`Sat`? `Cyclic`/`Bounded`?)
- [ ] **v0 priority** — Ship in v0, or defer until const generics?
