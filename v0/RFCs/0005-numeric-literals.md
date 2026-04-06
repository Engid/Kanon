# RFC-0005: Numeric Literals, Suffixes, and Unit Types

**Status:** Draft  
**Created:** 2026-04-05  
**Related:** §2.4 Numeric Literals, §3.1 Numeric Tower, §3.2 Music Types, §6.3 Formal Grammar

---

## Summary

Define how Clef handles numeric literal syntax — rationals, EDO steps, unit suffixes (`hz`, `c`), and which mechanisms are user-definable. This RFC proposes:

1. **`Int / Int → Ratio`** baked into the language (math-correct default)
2. **Drop `\` as an operator** — EDO uses `edo(7, 12)` or `7.edo(12)` via UFCS
3. **`unit` keyword** for branded numeric types with auto-derived arithmetic and literal suffixes
4. **`op_` convention** for user-defined operator overloading
5. **Scanner splits `<number><alpha>`** — suffix resolution tries unit type match, then function fallback

---

## Motivation

### The Core Question

How much of Clef's numeric literal syntax should be baked into the language grammar vs. pushed into the type system or library?

The original spec (§2.4, §6.3) treats `3/2` (rational) and `7\12` (EDO) as **grammar-level special cases** in the parser, and `440.0hz` / `701.955c` as **UFCS calls** to conversion functions. This RFC explores alternatives and proposes a unified approach.

---

## Design Exploration

### Approach 1: Grammar-Level Literals (Original Spec)

The parser recognizes `3/2` as a `RatioLiteral` and `7\12` as an `EdoLiteral` by checking that both operands of `/` or `\` are integer literals. `440.0hz` is UFCS for `hz(440.0)`.

**Pros:** Simple to implement. Syntax highlighting works at parse time.  
**Cons:** Special parser cases for `/` and `\`. Variables and literals behave differently. Not user-extensible.

### Approach 2: Operator Type Dispatch (No Special Parser Cases)

Remove the parser special cases. `/` and `\` are regular infix operators. The type system determines the result:

- `Int / Int → Ratio` (always — use `div()` for integer division)
- `Int \ Int → Edo`

**Pros:** Variables and literals follow the same rules. More consistent. Removes parser special cases.  
**Cons:** Syntax highlighting needs type info to distinguish. Requires type checker knowledge of operator rules.

**Conclusion:** Approach 2 is better. More consistent, fewer special cases. The parser just sees `BinaryExpr(3, /, 2)`; the type system resolves it to Ratio.

### Approach 3: Postfix Suffixes as UFCS

`440.0hz` is parsed as `440.0` `.` `hz` and resolved via UFCS to `hz(440.0)`.

**Pros:** User-extensible. No new language feature.  
**Cons:** Ambiguity between suffix and variable. The dot-less form `440.0hz` looks like a single token but isn't.

### Approach 4: User-Defined Literal Suffixes (Dedicated Feature)

A `suffix` keyword for defining what happens when a suffix appears on a numeric literal:

```
suffix hz(value: Float): Hz { Hz(value) }
suffix c(value: Float): Cents { Cents(value) }
```

**Pros:** Explicit, unambiguous, user-extensible.  
**Cons:** New keyword. Only works on literals. Overlaps with conversion functions.

### Approach 5: Hardcoded Suffixes (Ruby Model)

Bake a fixed set of suffixes into the scanner: `r`, `i`, `hz`, `c`.

**Pros:** Simplest. No ambiguity.  
**Cons:** Not user-extensible. Violates Steele's growability principle.

### Approach 6: Convention-Named Functions (Everything-Is-UFCS)

Unify operator overloading AND literal suffixes under the same mechanism: specially-named free functions.

#### Operator Overloading via `op_` Functions

```
fun op_div(self: Int, other: Int): Ratio = Ratio(self, other)
fun op_add(self: Music, other: Music): Music = Music.Seq(self, other)
```

**Operator name mapping:**

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

#### Literal Suffixes via `literal_suffix_` Functions

```
fun literal_suffix_hz(self: Float): Hz = Hz(self)
fun literal_suffix_r(self: Int): Ratio = Ratio(self, 1)
```

**Pros:** No new keywords. Both operators and suffixes use the same mechanism. Scoped to imports. User-extensible. Steele-growable.  
**Cons:** Verbose prefix. Encoding suffix in function name is unconventional.

### Approach 7: Unit Types — Type Declaration as Primary Mechanism

#### Option 7a: `unit` Keyword (FAVORED)

A dedicated `unit` declaration creates a branded numeric type with auto-derived arithmetic AND a literal suffix:

```
unit Hz = Float                     // branded Float, suffix 'hz' auto-registered
unit Cents = Float as "c"           // branded Float, explicit suffix 'c'
unit Bpm = Float                    // suffix 'bpm'

440.0hz         // → Hz(440.0)
701.955c        // → Cents(701.955)
120.0bpm        // → Bpm(120.0)
```

**Auto-derived arithmetic:**

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

**Suffix resolution:** Lowercase type name by default. If the name doesn't match (e.g., `Cents` → `c`), use `as "c"` to specify.

**Unwrap/wrap:** `Float(myHz)` unwraps explicitly. `Hz(440.0)` wraps explicitly. No implicit conversion.

---

## Prior Art

### F# Units of Measure

```fsharp
[<Measure>] type hz
440.0<hz>           // float<hz>
```

Compile-time dimension tracking with algebraic simplification. Doesn't directly apply to Clef — Clef needs representation changes (Ratio is a BigInt pair, not a tagged float).

### Ruby Rational Literals

`3r` is a hardcoded scanner token. `r` and `i` are the only suffixes. `2/3r` works via operator dispatch (`Integer / Rational → Rational`).

### C++ User-Defined Literals

```cpp
constexpr Hz operator""_hz(long double val) { return Hz{val}; }
auto freq = 440.0_hz;
```

User-extensible but syntactically awkward. Works only on literals.

### Nim Distinct Types

```nim
type Dollar = distinct int
```

Distinct types with user-definable suffixes via proc naming.

---

## Proposed Approach (Decisions Made)

### 1. `Int / Int → Ratio` — BAKED IN

This is math-correct, not music-specific. Racket/Scheme/Haskell do this. Use `div()` for integer division. `let a = 3; let b = 2; a / b` produces `Ratio(3, 2)`, same as the literal `3/2`.

### 2. `\` Operator — DROPPED

Not used as an infix operator in any mainstream language (except Julia/MATLAB matrix division). Too confusing with escape sequences. EDO uses function call or UFCS:

```
let fifth = edo(7, 12)       // function call
let fifth = 7.edo(12)        // UFCS — reads "7 edo 12"
```

### 3. `unit` Keyword — YES

One-line declaration for branded numeric types with auto-derived arithmetic and literal suffix:

```
unit Hz = Float

// That's it. 440.0hz works because:
// - scanner splits into (440.0, "hz")
// - resolver finds unit type Hz (suffix match)
// - wraps as Hz(440.0)
// - arithmetic auto-derived
```

### 4. `op_` Convention — YES

User-defined operators via `op_div`, `op_add`, etc. The numeric tower operators are defined this way in stdlib.

### 5. `r` Suffix — OPTIONAL, IN STDLIB

Defined as `fun r(x: Int): Ratio = Ratio(x, 1)`. Nice for `[3r, 5r, 7r]` but not essential since `3/1` works.

---

## Where Things Live

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

---

## Open Questions

1. **Suffix naming verbosity:** `op_div` vs `operator_div` vs `operator/`? Shorter is better since these appear frequently in stdlib code.

2. **Suffix on function name vs. parameter:** Instead of `literal_suffix_hz(self: Float)`, could it be `literal_suffix(self: Float, suffix: "hz")`? Requires string literal types.

3. **Operator precedence:** If operators are user-defined functions, who controls precedence? The language still needs a hardcoded precedence table. User-defined operators inherit fixed precedence from the operator symbol.

4. **`unit` vs `distinct`:** Should there also be a lower-level `distinct` keyword for non-numeric branding (e.g., `type SQL = distinct String`)? `unit` would be sugar over `distinct` + auto-generated operators.

5. **Clef as general-purpose:** The design leans toward fewer baked-in music concepts. Hz, Cents, Edo should be stdlib types, not language primitives. Ratio stays as a language-level numeric type (it's math, not music).

---

## Decisions Requested

- [ ] **Confirm `Int / Int → Ratio`** — Baked-in exact division as default?
- [ ] **Confirm `\` dropped** — EDO via `edo()` / UFCS only?
- [ ] **`unit` keyword** — Approve as core language feature?
- [ ] **`op_` naming** — Confirm `op_` prefix convention for operator overloading?
- [ ] **`distinct` keyword** — Add alongside `unit` for non-numeric branding?
- [ ] **v0 priority** — Which pieces ship in v0?
