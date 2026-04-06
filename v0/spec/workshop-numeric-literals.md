# Workshop: Numeric Literals, Suffixes, and Unit Types

> **Created:** 2026-04-05  
> **Last Updated:** 2026-04-06  
> **Changelog:**
> - 2026-04-05 — Initial draft
> - 2026-04-06 — Converted to RFC-0005; added metadata headers

**Status:** Open discussion. Not decided. Captures design exploration around how Clef handles numeric literal syntax — specifically rationals, EDO steps, unit suffixes (hz, c), and whether any of these should be user-definable.

> **Note:** This workshop document has been converted to [RFC-0005](../RFCs/0005-numeric-literals.md). Future updates should be made to the RFC.

---

## The Core Question

How much of Clef's numeric literal syntax should be baked into the language grammar vs. pushed into the type system or library?

Currently the spec (§2.4, §6.3) treats `3/2` (rational) and `7\12` (EDO) as **grammar-level special cases** in the parser, and `440.0hz` / `701.955c` as **UFCS calls** to conversion functions. This workshop explores alternatives.

---

## Approach 1: Grammar-Level Literals (Current Spec)

The parser recognizes `3/2` as a `RatioLiteral` and `7\12` as an `EdoLiteral` by checking that both operands of `/` or `\` are integer literals. `440.0hz` is UFCS for `hz(440.0)`.

**Pros:**
- Simple to implement — parser handles it
- Syntax highlighting can distinguish rationals from division at parse time
- No new language features needed

**Cons:**
- Special cases in the parser for `/` and `\`
- `let a = 3; let b = 2; a/b` behaves differently from `3/2` — variables don't get the literal treatment
- Not user-extensible

## Approach 2: Operator Type Dispatch (No Special Parser Cases)

Remove the parser special cases. `/` and `\` are regular infix operators. The type system determines the result:

- `Int / Int → Ratio` (always — use `div()` for integer division)
- `Int \ Int → Edo`
- `Ratio + Ratio → Ratio`
- `anything + Float → Float` (inexactness taints)

**Pros:**
- Variables and literals follow the same rules: `a / b` and `3/2` both produce Ratio when both operands are Int
- Removes parser special cases
- More consistent and predictable

**Cons:**
- Syntax highlighting needs type info to distinguish `3/2` (ratio) from `x / y` (could be anything) — though in practice Int/Int is always Ratio under this model
- Requires the type checker or interpreter to know operator type rules (it will anyway)

**Conclusion from discussion:** Approach 2 is likely better. More consistent, fewer special cases. The parser just sees `BinaryExpr(3, /, 2)`; the type system resolves it to Ratio.

## Approach 3: Postfix Suffixes as UFCS (Current Spec for hz/c)

`440.0hz` is parsed as `440.0` `.` `hz` and resolved via UFCS to `hz(440.0)`. Any function of one argument could work as a suffix.

**Pros:**
- User-extensible: `fun bpm(n: Float): Tempo = Tempo(n)` → `120.0bpm` works
- No new language feature — just UFCS

**Cons:**
- Requires a scanner rule to split `440.0hz` into `440.0` + `.` + `hz` (or `440.0` + `hz`)
- Ambiguity: is `hz` a variable or a suffix? What about `let hz = 5; 440.0hz`?
- The dot-less form `440.0hz` looks like a single token but isn't
- `440.0.hz` (with dot) has its own ambiguity: is `.` a decimal point or member access?

**Partially resolved:** The scanner can split on the number-to-alpha boundary without whitespace. But edge cases remain.

## Approach 4: User-Defined Literal Suffixes (Dedicated Feature)

A language feature specifically for defining what happens when a suffix appears on a numeric literal. Like C++ user-defined literals but with cleaner syntax.

**Hypothetical syntax:**
```
suffix hz(value: Float): Hz { Hz(value) }
suffix c(value: Float): Cents { Cents(value) }
suffix i(value: number): Complex { Complex(0, value) }
suffix r(value: Int): Ratio { Ratio(value, 1) }
```

**Then:**
```
440.0hz       // invokes hz suffix → Hz(440.0)
701.955c      // invokes c suffix → Cents(701.955)
3i            // invokes i suffix → Complex(0, 3)
2r            // invokes r suffix → Ratio(2, 1)
```

**Pros:**
- Explicit — `suffix` keyword makes intent clear
- Unambiguous — suffixes are a distinct namespace, no collision with variables
- User-extensible — library authors can define new ones
- Only works on literals, not variables — clean separation of concerns
- Steele/Lattner "growability" principle: builtins and user-defined are indistinguishable

**Cons:**
- New keyword and declaration form in the grammar
- Only works on literals — for variables you'd use the normal function: `hz(freq)`
- Somewhat overlaps with just having conversion functions
- Scope: which literal types can have suffixes? Just numbers? Strings? Objects?

**Open question:** Is the extra language feature worth it, or is baking a few suffixes into the scanner (like Ruby does with `r` and `i`) simpler and good enough?

## Approach 5: Hardcoded Suffixes (Ruby Model)

Bake a small fixed set of suffixes into the scanner: `r`, `i`, `hz`, `c`. Not user-extensible. The scanner recognizes `3r` as a Ratio literal, `2i` as imaginary, `440.0hz` as Hz, `701.955c` as Cents.

**Pros:**
- Simplest to implement — all in the scanner
- No ambiguity, no edge cases
- Syntax highlighting just works

**Cons:**
- Not user-extensible — adding a new suffix requires changing the language
- Violates Steele's growability principle

---

## How F# Does Unit Types (Reference)

F# has a first-class **units of measure** system. It's a compile-time language feature, not a library:

```fsharp
[<Measure>] type hz
[<Measure>] type cm
[<Measure>] type s
[<Measure>] type N = kg m / s^2    // derived units with algebra

440.0<hz>           // float<hz> — the angle brackets annotate the unit
9.8<m/s^2>          // the compiler tracks compound units
```

Key properties:
- `float<hz>` and `float<cm>` are **different types** — can't add them
- Arithmetic **automatically computes unit types**: `distance / time` infers `float<m/s>`
- Compound units with algebraic simplification: `m/s * s = m`
- Units are **completely erased at runtime** — zero overhead
- Generic functions can be polymorphic over units: `fun add(x: float<'u>, y: float<'u>)`

**Why it doesn't directly apply to Clef:** F# units tag a float with dimensional metadata. They work for multiplicative dimensional analysis (meters, seconds, kg). Clef's types are fundamentally different:
- `Ratio` is a different *representation* (exact BigInt pair), not a tagged float
- `Edo` carries a runtime value (the division count: 12, 19, 31) — not a compile-time dimension
- `Cents` and `Hz` don't compose dimensionally — `Hz * Hz` isn't meaningful
- Clef needs conversions that change representation: `cents(3/2)` computes a logarithm

F# units answer "is this meters or seconds?" Clef types answer "is this exact or approximate?" — a deeper semantic distinction.

---

## How Ruby Does Rational Literals (Reference)

`3r` is a **single token** in Ruby's lexer — hardcoded, not user-extensible. `r` and `i` are the only two suffixes.

`1/2r` works as: `1 / 3r` → `Integer / Rational → Rational`. The "whole expression becomes rational" behavior comes from operator dispatch, not literal syntax.

```ruby
12r          # => Rational(12, 1)
12.3r        # => Rational(123, 10)
2/3r         # => Rational(2, 3)    — because 3r is Rational(3,1), and Int/Rational → Rational
1i           # => Complex(0, 1)
12.3ri       # => Complex(0, Rational(123, 10))
```

---

## How C++ Does User-Defined Literals (Reference)

```cpp
constexpr Hz operator""_hz(long double val) { return Hz{val}; }
constexpr Cents operator""_c(long double val) { return Cents{val}; }

auto freq = 440.0_hz;     // works
auto pitch = 701.955_c;   // works
```

User-extensible but syntactically ugly. The `std::chrono` literals (`100ms`, `2s`, `30min`) are the most-used example. Works only on literals, not expressions.

---

## Approach 6: Convention-Named Functions (Everything-Is-UFCS)

Unify operator overloading AND literal suffixes under the same mechanism: specially-named free functions, resolved via the same scoping rules as UFCS. No new keywords. No new declaration forms. Just naming conventions that the language recognizes.

### Operator Overloading via Named Functions

Functions whose names start with `op_` are treated as operator implementations. The language maps operators to function names:

```
// defining / for Rational types
fun op_div(self: Int, other: Int): Ratio = Ratio(self, other)
fun op_div(self: Ratio, other: Int): Ratio = ...
fun op_div(self: Ratio, other: Ratio): Ratio = ...
fun op_div(self: Int, other: Ratio): Ratio = ...

// defining \ for EDO
fun op_backslash(self: Int, divisions: Int): Edo = Edo(self, divisions)

// defining + for Music types
fun op_add(self: Music, other: Music): Music = Music.Seq(self, other)
```

When the interpreter/checker encounters `3 / 2`, it looks up `op_div(Int, Int)` in scope. Same dispatch rules as UFCS — the function must be imported to be available.

**Operator name mapping (tentative):**

| Operator | Function name |
|----------|--------------|
| `+` | `op_add` |
| `-` | `op_sub` |
| `*` | `op_mul` |
| `/` | `op_div` |
| `%` | `op_mod` |
| `\` | `op_backslash` |
| `==` | `op_eq` |
| `!=` | `op_neq` |
| `<` | `op_lt` |
| `<=` | `op_lte` |
| `>` | `op_gt` |
| `>=` | `op_gte` |
| `\|>` | `op_pipe` |

**Alternative naming:** Could use more verbose names (`operator_div`, `operator_add`) or symbolic names (`operator/`, `operator+` — if the grammar allows symbols in function names after `operator`). The `op_` prefix is short and clear. Python uses `__add__`, Kotlin uses `operator fun plus`, Rust uses `impl Add` trait. The `op_` prefix is closest to Python's dunder approach.

### Literal Suffixes via Named Functions

Functions whose names start with `literal_suffix_` define what happens when that suffix appears on a numeric literal. The suffix text is encoded in the function name:

```
// stdlib definitions
fun literal_suffix_r(self: Int): Ratio = Ratio(self, 1)
fun literal_suffix_i(self: number): Complex = Complex(0, self)
fun literal_suffix_hz(self: Float): Hz = Hz(self)
fun literal_suffix_c(self: Float): Cents = Cents(self)

// user-defined
fun literal_suffix_bpm(self: Float): Tempo = Tempo(self)
fun literal_suffix_db(self: Float): Decibels = Decibels(self)
```

**Then in code:**
```
3r              // scanner splits → resolver calls literal_suffix_r(3)
2i              // scanner splits → resolver calls literal_suffix_i(2)
440.0hz         // scanner splits → resolver calls literal_suffix_hz(440.0)
701.955c        // scanner splits → resolver calls literal_suffix_c(701.955)
120.0bpm        // scanner splits → resolver calls literal_suffix_bpm(120.0)
-6.0db          // scanner splits → resolver calls literal_suffix_db(6.0), then negate
```

**How Ruby-style `1/2r` would work under this model:**
1. Scanner splits `2r` into `INT(2)` + suffix `r`
2. Resolver calls `literal_suffix_r(2)` → `Ratio(2, 1)`
3. Expression becomes `1 / Ratio(2, 1)`
4. `op_div(Int, Ratio)` → `Ratio(1, 2)`

The rational behavior falls out of operator dispatch + suffix resolution. Clean.

### Properties of This Approach

**Pros:**
- No new keywords — just naming conventions on regular `fun` declarations
- Both operators and suffixes use the same mechanism (named functions)
- Scoped to imports — like UFCS, an operator or suffix only activates if its defining function is in scope. This prevents "phantom operator" surprises from transitive dependencies
- User-extensible — library authors define new operators/suffixes identically to stdlib
- Steele's growability: user-defined `120.0bpm` is indistinguishable from builtin `440.0hz`
- The stdlib's numeric tower is defined in the stdlib, not hardcoded in the language

**Cons:**
- `literal_suffix_` prefix is verbose (but only in the definition — users never see it)
- Encoding the suffix in the function name is unconventional (though Python's dunder methods establish precedent)
- The scanner still needs a grammar-level rule: "split `<number><alpha>` into two tokens." The naming convention determines resolution, not parsing
- Multiple overloads may be needed for each operator (Int×Int, Int×Ratio, Ratio×Ratio, etc.)
- The `op_` prefix could collide with user names that happen to start with `op_` (mitigated: `op_` + known operator name is a finite set)

### Open Design Questions

1. **Naming verbosity:** `op_div` vs `operator_div` vs `operator/` vs something else? Shorter is better since these appear frequently in stdlib code.

2. **Suffix on the function name vs. as a parameter:** Instead of `literal_suffix_hz(self: Float)`, could it be `literal_suffix(self: Float, suffix: "hz")` with a string literal type parameter? This is more conventional but requires string literal types or comptime strings.

3. **Prefix literals too?** Should there be a `literal_prefix_` convention? (e.g., for negative literals, or hypothetical `#FF0000` color literals). Probably not needed for v0.

4. **Operator precedence:** If operators are user-defined functions, who controls precedence? The language still needs a hardcoded precedence table for `+`, `*`, `/`, etc. User-defined operators would need a way to specify precedence (or just inherit fixed precedence from the operator symbol they overload).

5. **Discoverability:** How does a user know what suffixes are available? IDE tooling could list all `literal_suffix_*` functions in scope. But that requires knowing the convention.

---

## Approach 7: Unit Types — Type Declaration as the Primary Mechanism

### The Core Insight

The suffix on `440.0hz` isn't really a "function call." It's a **branding** — it says "this number is an Hz, not a bare Float." The question: should the definition of `hz` be a type declaration, not a function?

This reframes the problem: **user-defined units are user-defined types**, and the literal suffix is just sugar for constructing them.

### Research: How Languages Handle Branded Numeric Types

| Language | Mechanism | Declaration | User-extensible suffixes |
|----------|-----------|------------|--------------------------|
| **Nim** | `distinct` keyword | `type Dollar = distinct int` | Yes: proc `` `'suffix` ``(n: string): T` |
| **Ada** | Derived types | `type Meters is new Float;` | No — explicit conversions |
| **Scala 3** | `opaque type` | `opaque type Hz = Double` | No — companion `.apply()` |
| **Kotlin** | `value class` | `value class Hz(val v: Float)` | No — explicit constructor |
| **F#** | `[<Measure>]` | `[<Measure>] type hz` → `440.0<hz>` | Yes — but angle-bracket syntax |
| **Haskell** | `newtype` | `newtype Hz = Hz Float` | No — constructor |
| **Rust** | Newtype pattern | `struct Hz(f64);` | No — `Hz(440.0)` |

**Observation:** Almost no language supports postfix literal suffixes as a user-definable type feature. F# is closest (ties units to the type system) but uses angle-bracket syntax. Nim has user-definable literal suffixes but via function naming, not type declarations.

### Option 7a: `unit` Keyword (FAVORED)

A dedicated `unit` declaration creates a branded numeric type with auto-derived arithmetic AND a literal suffix. The compiler generates dimensionally sensible operators so you never have to borrow/derive them manually.

```
unit Hz = Float                     // branded Float, suffix 'hz' auto-registered
unit Cents = Float as "c"           // branded Float, explicit suffix 'c'
unit Bpm = Float                    // suffix 'bpm'

440.0hz         // → Hz(440.0)
701.955c        // → Cents(701.955)
120.0bpm        // → Bpm(120.0)
```

**Auto-derived arithmetic:** The compiler generates operators following dimensional rules:

| Expression | Result | Rule |
|-----------|--------|------|
| `Hz + Hz` | `Hz` | Same-unit addition |
| `Hz - Hz` | `Hz` | Same-unit subtraction |
| `Hz * Float` | `Hz` | Scaling by dimensionless |
| `Float * Hz` | `Hz` | Scaling by dimensionless |
| `Hz / Float` | `Hz` | Scaling by dimensionless |
| `Hz / Hz` | `Float` | Same-unit ratio is dimensionless |
| `Hz + Float` | **ERROR** | Can't mix branded and unbranded |
| `Hz + Cents` | **ERROR** | Can't mix different units |
| `Hz * Hz` | **ERROR** | Not defined (user can add it) |
| `-Hz` | `Hz` | Unary negation |
| `Hz == Hz` | `bool` | Comparison |
| `Hz < Hz` | `bool` | Ordering |

The user writes one line. The compiler does the rest. If you want custom behavior (e.g., `Hz * Duration → Cycles`), write an explicit `op_mul`.

**Suffix resolution:** The suffix is the lowercase type name by default. If the name doesn't match (e.g., `Cents` → `c`), use `as "c"` to specify an explicit suffix.

**Unwrap/wrap:** `Float(myHz)` unwraps explicitly. `Hz(440.0)` wraps explicitly. No implicit conversion.

**Pros:**
- **One line** gives you: type safety, literal suffix, arithmetic, comparison
- No borrow pragmas, no boilerplate operator definitions
- The `unit` keyword communicates intent clearly
- The compiler-generated arithmetic follows standard dimensional analysis rules
- User can override any auto-derived operator with an explicit definition

**Cons:**
- New keyword (`unit`) — but it's small and specific
- Only works for numeric brands — complex types like Ratio still need manual construction
- The suffix name rule needs to be clear (lowercase type name, or explicit `as`)
- Doesn't cover non-numeric distinct types (but `distinct` could exist separately for that)

### Option 7b: `distinct` Types + Automatic Suffix Resolution

Add `distinct` as a type modifier (like Nim/Ada). The suffix mechanism is separate: when the scanner produces `NumberLiteral + UnitName`, the resolver tries to find a **type constructor** with that name:

```
type Hz = distinct Float
type Cents = distinct Float
type Bpm = distinct Float

// 440.0hz → scanner emits (440.0, "hz")
//        → resolver looks for type "Hz" (case-insensitive match)
//        → calls Hz(440.0) — the distinct type's constructor

// 701.955c → looks for type "C"? NOT FOUND
//         → then looks for function "c" in scope
//         → falls back to function resolution
fun c(x: Float): Cents = Cents(x)   // needed because "c" ≠ "Cents"

// For nontrivial conversion:
fun r(x: Int): Ratio = Ratio(x, 1)  // Ratio isn't a simple brand
```

**Resolution order for suffix `xyz` on a number:**
1. Look for a type named `Xyz` (or `xyz`, case-insensitive) that is `distinct` over a compatible numeric type → call its constructor
2. If not found, look for a function named `xyz` that accepts the number type → call it
3. If neither, error

**Pros:**
- `distinct` is a useful general-purpose feature beyond just units
- Simple cases (Hz, Bpm) need zero extra code — just the type declaration
- Complex cases (Ratio, Cents with short suffix) fall back to functions
- Two useful features (`distinct` types + suffix resolution) instead of one single-purpose feature

**Cons:**
- Two-step resolution adds complexity
- The case-insensitive type lookup (`hz` → `Hz`) may surprise
- Need to define what arithmetic `distinct` types inherit (see below)

### Option 7c: `distinct` Types + Explicit Suffix Annotation

Like 7b, but the suffix name is explicitly declared on the type:

```
type Hz = distinct Float        // no automatic suffix
type Cents = distinct Float

suffix hz = Hz                  // "440.0hz constructs an Hz"
suffix c = Cents                // "701.955c constructs a Cents"
suffix r(x: Int) = Ratio(x, 1) // "3r calls this function"
suffix bpm = Bpm
```

Or, integrated into the type declaration:

```
type Hz = distinct Float with suffix "hz"
type Cents = distinct Float with suffix "c"
type Bpm = distinct Float with suffix "bpm"
```

**Pros:**
- Explicit mapping — no guessing about case conventions
- Clean separation: the type is the type, the suffix is the suffix
- `suffix` declaration is simple and scannable

**Cons:**
- Yet another declaration form (`suffix`)
- The `with suffix` variant adds syntax to the type declaration

### What `unit` vs `distinct` Gives Clef

`unit` is the **happy path** for numeric types — it gives you arithmetic, suffixes, and type safety in one declaration. But the language could also have a lower-level `distinct` keyword for non-numeric branding (e.g., `type SQL = distinct string`) where auto-derived arithmetic doesn't make sense.

Relationship:
- `unit Hz = Float` — branded numeric type, auto-derives arithmetic, has literal suffix
- `type SQL = distinct string` — branded non-numeric type, inherits nothing, no suffix

`unit` is sugar over `distinct` + auto-generated operators. You get the happy path for numbers, and the general mechanism for everything else.

```
unit Hz = Float
unit Cents = Float as "c"

let a: Hz = 440.0hz
let b: Cents = 701.955c

a + b              // ERROR: can't add Hz and Cents
a + 20.0           // ERROR: can't add Hz and bare Float
a + 20.0hz         // OK — auto-derived Hz + Hz → Hz
a * 2.0            // OK — auto-derived Hz * Float → Hz

Float(a)           // explicit unwrap: 440.0
Hz(440.0)          // explicit wrap
```

### Recommendation: `unit` keyword (Option 7a) + function fallback for non-unit suffixes

1. **`unit` keyword** for branded numeric types — one-line happy path with auto-derived arithmetic
2. **Literal suffix** comes from the unit type name (lowercase) or explicit `as` — no separate declaration needed
3. **Function fallback** for suffixes that don't map to a unit type (e.g., `r` for Ratio)
4. **`op_` convention** (from Approach 6) for custom operator overloading beyond the auto-derived set
5. **Scanner** splits `<number><alpha>` into two tokens — always

This makes the common case concise:
```
unit Hz = Float

// That's it. 440.0hz works because:
// - scanner splits into (440.0, "hz")
// - resolver finds unit type Hz (suffix match)
// - wraps as Hz(440.0)
// - arithmetic auto-derived
```

And the complex case has a clean fallback:
```
fun r(x: Int): Ratio = Ratio(x, 1)
// 3r works because: no unit type "R" found, falls back to function r(Int)
```

---

## On Baking Rationals and EDO into the Language

### Rationals (`3/2`)

**Should `Int / Int → Ratio` be baked in?**

Argument for baking it: This is just how math works. 3 divided by 2 is 1.5, or more precisely, the rational 3/2. Making `/` on integers produce a Ratio (exact) rather than a truncated Int (lossy) is arguably the *correct* default. Racket, Scheme, Common Lisp, and Haskell all do this. It's not a music-specific behavior — it's a math-correct behavior.

Argument against: In most general-purpose languages, `3 / 2 == 1` (integer division). Users from JS/Python/C expect this. Making it Ratio is surprising. Provide `div()` for integer division and `/` for exact division? That's what Clef does, and it's defensible, but it IS a choice that makes Clef less "general purpose" in feel.

**Current leaning:** Keep it. `Int / Int → Ratio` is arguably the "right" thing. It's a language design philosophy statement: "we don't lose precision silently." This is valuable for general-purpose use too, not just music. Racket proves this works in a broadly-used language.

### `r` Suffix (`3r → Ratio(3, 1)`)

**Is this needed?** If `Int / Int → Ratio` is baked in, then `3/1` already gives you `Ratio(3, 1)`. So `3r` is sugar for `3/1`. Is that worth a suffix?

Cases where `r` helps:
- `3r` is shorter than `3/1`
- It signals intent: "I want a Ratio, not an Int"
- Pattern: `[3r, 5r, 7r]` for a list of Ratios vs `[3/1, 5/1, 7/1]`

**Current leaning:** Nice-to-have, not essential. If distinct types + suffix resolution exists, `r` can be defined in stdlib as a function (`fun r(x: Int): Ratio = Ratio(x, 1)`). No need to bake it into the language.

### EDO — Dropping `\` as an Operator

**Decision: Drop `\` from the grammar entirely.**

`\` is not used as an infix operator in any mainstream language except Julia/MATLAB (matrix left division). In every other language it's reserved for escape sequences. Having it as an operator in Clef would:
- Confuse users who expect `\` = escape character
- Make syntax highlighting harder (is `7\12` an operator expression or a malformed escape?)
- Add a grammar-level feature for one domain-specific use case

**How EDO works without `\`:**

Edo becomes a normal type in the music stdlib, constructed via function call or UFCS:

```
// In music stdlib:
type Edo = { step: Int, divisions: Int }

fun edo(step: Int, divisions: Int): Edo = Edo { step, divisions }

// Usage:
let fifth = edo(7, 12)       // function call
let third = edo(6, 19)

// UFCS:
let fifth = 7.edo(12)        // reads "7 edo 12"
let minor_third = 3.edo(12)  // reads "3 edo 12"
```

`7.edo(12)` via UFCS reads quite naturally. And since Edo is just a struct in the music stdlib, non-music users never encounter it.

**Could EDO have a suffix?** `7edo12` doesn't work well — the suffix mechanism is `<number><alpha>`, which would split to `(7, "edo12")`. A two-argument suffix doesn't fit the model. Function call or UFCS is the right answer here.

**What about the spec's existing examples?** All `7\12` examples in the spec become `edo(7, 12)` or `7.edo(12)`. The UFCS form is nearly as concise and much less surprising.

---

## On Clef as a More General-Purpose Language

The user is gravitating toward "a more general purpose language that is hostable and lightweight to read and write." This shifts several design priorities:

1. **Fewer baked-in music concepts** — Hz, Cents, Edo should be stdlib types, not language primitives. Ratio stays as a language-level numeric type (it's math, not music).
2. **More general mechanisms** — `unit` types, operator overloading, suffix resolution serve ALL domains, not just music
3. **The music stdlib IS the killer app** — but the language shouldn't require it
4. **Hostability** — the ClefHost interface becomes even more important. A game engine, audio plugin, or CLI tool can all embed Clef

This means the core language should provide:
- A clean numeric tower (Int, Ratio, Float — with exact arithmetic)
- `unit` keyword for branded numeric types with auto-derived arithmetic
- `distinct` keyword for general-purpose type branding (non-numeric)
- User-defined operators via `op_` convention
- User-defined literal suffixes (via unit types + function fallback)
- UFCS, pattern matching, generators, algebraic types — all general-purpose features

And the music stdlib layers on:
- `unit Hz = Float`
- `unit Cents = Float as "c"`
- `type Edo = { step: Int, divisions: Int }`
- `fun edo(step: Int, divisions: Int): Edo`
- `type Music = Note(...) | Rest(...) | Seq(...) | Par(...) | Modify(...)`
- `fun realize(m: Music): list[Event]`
- Standard scales, temperaments, instruments

---

## Current Leaning (Updated — April 2026)

### Decisions Made

1. **`Int / Int → Ratio` — BAKED IN.** This is math-correct, not music-specific. Racket/Scheme/Haskell do this. Use `div()` for integer division.

2. **`\` operator — DROPPED.** Not used as an infix operator in any mainstream language (except Julia/MATLAB matrix division). Too confusing with escape sequences. EDO uses `edo(7, 12)` or `7.edo(12)` via UFCS.

3. **`unit` keyword — YES.** One-line declaration for branded numeric types with auto-derived arithmetic and literal suffix. `unit Hz = Float` gives you everything.

4. **`op_` convention — YES.** User-defined operators via `op_div`, `op_add`, etc. The numeric tower operators are defined this way in stdlib.

5. **`r` suffix — OPTIONAL, IN STDLIB.** Defined as `fun r(x: Int): Ratio = Ratio(x, 1)`. Nice for `[3r, 5r, 7r]` but not essential since `3/1` already works.

### Infix operators (`/`, `+`, `*`, etc.)

**Approach 2 + 6 combined:** Operators dispatch on argument types. The mechanism is `op_` naming convention (Approach 6). The stdlib defines the numeric tower operators. `op_div` is the right name for the division concept.

### Postfix suffixes (`hz`, `c`, `bpm`, `r`)

**Approach 7a (unit types) + function fallback**: The scanner always splits `<number><alpha>`. Resolution tries: (1) unit type by suffix match, (2) function by name. `unit` types provide the branding + arithmetic. Functions handle complex cases (like `r`).

### What lives where

| Feature | Where | Mechanism |
|---------|-------|-----------|
| `Int / Int → Ratio` | Language core | Built-in numeric tower |
| `div(Int, Int) → Int` | Language core | Built-in function |
| `unit` keyword | Language core | Type declaration |
| `op_` convention | Language core | Naming convention |
| `Hz`, `Cents`, `Bpm` | Music stdlib | `unit` declarations |
| `Edo` type | Music stdlib | Product type + `edo()` function |
| `r` suffix | Stdlib | `fun r(x: Int): Ratio` |
| `hz`, `c` suffixes | Music stdlib | Registered via `unit` declarations |
| `Music` algebraic type | Music stdlib | Sum type |
| `realize()` | Music stdlib | Function |

### For v0 implementation

Start with:
- `unit` type support (branded wrapper, auto-derived arithmetic, literal suffix registration)
- Hardcoded numeric tower operators in the interpreter (can migrate to `op_` later)
- Scanner rule: split `<number><alpha>` into two tokens
- Suffix resolution: unit type lookup first, then function lookup
- No `\` operator in the grammar

---

## Related Spec Sections

- §2.4 Numeric Literals
- §3.1 Numeric Tower
- §3.2 Music Types
- §6.3 Parsing Notes (rational/EDO literal disambiguation)
- §10 Sketch Notes (open questions)
- workshop-interfaces.md (related: trait/interface system could formalize operator overloading)
- workshop-naming.md (naming conventions for `op_` prefix)

## Cross-References from Research

- **Nim `distinct` type**: [Manual §Distinct type](https://nim-lang.org/docs/manual.html#types-distinct-type) — modeling currencies, SQL injection prevention, `borrow` pragma for arithmetic
- **Nim custom numeric literals**: `123'suffix` syntax, proc `` `'suffix` ``(n: string): T`
- **Ada derived types**: `type Meters is new Float;` — incompatible with Float, explicit conversion
- **Scala 3 opaque types**: `opaque type Hz = Double` — transparent inside defining scope, opaque outside
- **Kotlin value classes**: `value class Hz(val v: Float)` — zero-overhead wrapper, can implement interfaces
- **F# units of measure**: `[<Measure>] type hz` — compile-time dimensional algebra, erased at runtime
