# RFC: Unit Types for Numeric Values

**Status:** Draft  
**Created:** 2026-04-09  
**Feature:** `unit` keyword for defining measurement-annotated numeric types  
**Depends on:** Core numeric types (Int, Float, Ratio), type system basics  
**Supersedes:** This RFC is the source of truth for the unit type feature. Where other documents (RFC-0005, §2.4, §3.2, §6.3) describe unit-related behavior that conflicts with this RFC, this RFC takes precedence.

---

## Summary

Introduce a `unit` keyword that creates distinct numeric types representing physical or musical measurements (Hz, Cents, Beats, etc.). Unit types are transparent wrappers — a `Hz` value IS a `Float` at runtime — but the type checker treats them as distinct types, preventing accidental mixing of incompatible measurements.

The feature is designed in levels, each building on the previous. Level 0 (simple named units) ships first. Level 1 (unit-polymorphic functions) and Level 2 (derived units) follow as the type system matures.

## Motivation

Kanon is a language for microtonal music composition and mathematical/DSP work where numerical precision and correctness matter. Confusing a frequency (Hz) with an interval (Cents), or accidentally adding a pitch ratio to a beat duration, is a real source of bugs in music and audio programming.

```
// dangerous: what do these numbers mean?
let x = 440.0
let y = 700.0
let z = x + y   // 1140.0 — but adding Hz to Cents is meaningless
```

With unit types:

```
unit Hz = Float
unit Cents = Float

let x = 440.0Hz
let y = 700.0Cents
let z = x + y   // ERROR: cannot add Hz and Cents
```

Beyond music, Kanon aims to support physics calculations, physical modeling synthesis, and general mathematical DSP — all domains where dimensional correctness matters.

## Three Axes of Numeric Values

Every numeric value in Kanon has three orthogonal properties:

**Mathematical classification (number tower):** What set of numbers does this belong to? Integer → Rational → Real → Complex. The `i` suffix on a literal moves a value from Real to Complex.

**Unit (what it measures):** What physical/musical quantity does this number represent? Hz, Cents, Beats, Meters, Seconds, Kilograms, etc. This is metadata about meaning.

**Computational representation:** How is this number stored? Int, Float, Ratio, BigInt. This affects precision and performance.

A value like `440.0Hz` sits at the intersection: Real (math), Hz (unit), Float (representation).

The `i` suffix and unit suffixes are fundamentally different — `i` changes the mathematical nature of the number, unit names label what it measures. They don't combine on the same literal.

---

## Level 0: Simple Named Units

### Declaration

```
unit <PascalCaseName> = <BaseType> [, <BaseType>]*
```

```
unit Hz = Float, Int
unit Cents = Float
unit Beats = Ratio, Int, Float
unit Samples = Int
unit BPM = Float, Int
unit Decibels = Float
```

### Literal Construction

A unit name (PascalCase, matching the declaration exactly) follows a numeric literal. Whitespace is optional.

```
let freq = 440.0Hz
let also_freq = 440.0 Hz       // space is fine
let midi_freq = 440Hz           // Hz (backed by Int)
let interval = 700.0Cents
let duration = 3/2 Beats        // Beats (backed by Ratio)
```

Case-sensitive — must match the declaration:

```
let a = 440Hz       // OK
let b = 440hz       // ERROR: 'hz' is not a defined unit
```

### Auto-Generated Arithmetic

Given `unit Hz = Float, Int`, the compiler generates:

**Same-unit addition and subtraction:**

```
Hz + Hz → Hz
Hz - Hz → Hz
```

```
let a = 440.0Hz + 20.0Hz     // 460.0Hz
```

**Scalar multiplication and division (dimensionless operand):**

```
Hz * Float → Hz
Float * Hz → Hz
Hz / Float → Hz
```

```
let a = 440.0Hz * 2.0        // 880.0Hz
let b = 2.0 * 440.0Hz        // 880.0Hz
```

**Same-unit division (produces dimensionless scalar):**

```
Hz / Hz → Float
```

```
let ratio = 880.0Hz / 440.0Hz   // 2.0 (dimensionless)
```

**Negation and comparison:**

```
-440.0Hz                         // -440.0Hz
440.0Hz < 880.0Hz               // true
440.0Hz == 440.0Hz              // true
```

**Cross-unit arithmetic is an error:**

```
440.0Hz + 700.0Cents            // ERROR: cannot add Hz and Cents
440.0Hz * 700.0Cents            // ERROR: cannot multiply Hz by Cents
```

### Dimensionless Arithmetic

A dimensionless (untagged) value can participate in arithmetic with a unit value **if they share the same representation type.** The dimensionless value is treated as a scalar.

```
let a = 440.0Hz + 20.0         // OK: Hz + Float → Hz
let b = 440.0Hz * 2.0          // OK: Hz * Float → Hz
let c = 440Hz + 20             // OK: Hz + Int → Hz
```

When the representations differ, an explicit cast is required:

```
let d = 440.0Hz + 20           // ERROR: Hz (Float-backed) + Int — mismatched representation
let e = 440.0Hz + Float(20)    // OK: cast Int to Float first
```

This rule is simple: same representation → dimensionless arithmetic is fine. Different representation → explicit cast. The unit system doesn't override the normal numeric promotion rules — it layers on top of them.

### Explicit Conversion

**Stripping a unit:** Use the base type as a function:

```
let raw = Float(440.0Hz)        // 440.0 (dimensionless)
```

**Adding a unit:** Use the unit name or literal suffix:

```
let freq = Hz(440.0)            // 440.0Hz
let freq = 440.0Hz              // same thing
```

**Between units:** User-defined conversion functions:

```
fun hz_to_cents(f: Hz, ref: Hz): Cents {
    1200.0Cents * log2(f / ref)
}
```

### Printing

Unit values auto-print with their unit annotation:

```
print(440.0Hz)          // output: 440.0Hz
print(700.0Cents)       // output: 700.0Cents
print(3/2 Beats)        // output: 3/2Beats
```

The unit name is preserved at runtime for display purposes. This is a small exception to the "zero-cost erasure" principle — the unit tag is erased for arithmetic but retained for string conversion.

---

## Level 1: Unit-Polymorphic Functions

### Motivation

Without unit polymorphism, generic numeric utilities must be duplicated per unit:

```
fun double_hz(x: Hz): Hz = x * 2.0
fun double_cents(x: Cents): Cents = x * 2.0
// ... tedious
```

### Syntax

A unit variable appears as a generic parameter constrained to a representation type:

```
fun double[U : Float](x: U): U = x * 2.0
```

`U : Float` means "U is any unit type whose base representation is Float." The function works on `Hz`, `Cents`, `Decibels`, or any other Float-based unit — including dimensionless `Float`.

```
double(440.0Hz)         // 880.0Hz
double(700.0Cents)      // 1400.0Cents
double(3.14)            // 6.28 (dimensionless Float)
```

### Multi-Parameter Examples

```
// linear interpolation — works on any Float-based unit
fun lerp[U : Float](a: U, b: U, t: Float): U =
    a + (b - a) * t

lerp(440.0Hz, 880.0Hz, 0.5)     // 660.0Hz
lerp(0.0Cents, 1200.0Cents, 0.5) // 600.0Cents

// clamp — both bounds must be the same unit
fun clamp[U : Float](x: U, lo: U, hi: U): U =
    if x < lo { lo } else if x > hi { hi } else { x }

clamp(50.0Hz, 20.0Hz, 20000.0Hz)   // 50.0Hz
```

### Multi-Representation Unit Variables

A unit variable can be constrained to multiple representations:

```
// works on Int or Float based units
fun double[U : Float, Int](x: U): U = x * 2
```

Or unconstrained (works on any representation):

```
fun identity[U](x: U): U = x
```

### Implementation

The type checker treats unit variables like generic type variables but in the unit dimension. When `double[U : Float](x: U): U` is called with `440.0Hz`, the checker unifies `U = Hz` and verifies that Hz has Float as a base representation.

Internally, unit variables are represented as unknowns in the exponent map (see Level 2). At Level 1, the exponent map is always a single entry `{U: 1}`, so unification is simple.

---

## Level 2: Derived Units

### Motivation

Many useful quantities are combinations of base units:

- Velocity = distance / time
- Sample rate = samples / time
- Tempo = beats / time
- Force = mass × acceleration
- Pressure = force / area

Without derived units, these relationships can only be expressed through manual conversion functions. With derived units, the type checker verifies dimensional consistency automatically.

### Declaration Syntax

Derived units are declared in terms of other units using `*`, `/`, and `^`:

```
// base units
unit Meters = Float
unit Seconds = Float
unit Kilograms = Float
unit Samples = Int, Float
unit Beats = Ratio, Float

// derived units
unit MetersPerSecond = Meters / Seconds
unit Acceleration = Meters / (Seconds ^ 2)
unit Newtons = Kilograms * Meters / (Seconds ^ 2)
unit Pascals = Newtons / (Meters ^ 2)

// music/audio derived units
unit SampleRate = Samples / Seconds
unit Tempo = Beats / Seconds
```

### Automatic Derivation from Arithmetic

When you perform arithmetic on unit values, the compiler tracks and combines the exponent maps:

```
let distance = 100.0Meters
let time = 9.58Seconds
let speed = distance / time
// compiler infers: MetersPerSecond
// because {Meters: 1} / {Seconds: 1} = {Meters: 1, Seconds: -1}
// which matches the declaration of MetersPerSecond
```

```
let rate = 48000.0 Samples / 1.0Seconds
// compiler infers: SampleRate
// {Samples: 1, Seconds: -1} matches SampleRate declaration

let duration = 2.5Seconds
let num_samples = rate * duration
// {Samples: 1, Seconds: -1} * {Seconds: 1} = {Samples: 1}
// compiler infers: Samples — the Seconds cancel
```

### Exponent Algebra

Internally, each unit-annotated type carries an exponent map: a mapping from base unit names to integer exponents.

| Value | Exponent Map |
|-------|-------------|
| `100.0Meters` | `{Meters: 1}` |
| `9.58Seconds` | `{Seconds: 1}` |
| `speed` (Meters/Seconds) | `{Meters: 1, Seconds: -1}` |
| `accel` (Meters/Seconds²) | `{Meters: 1, Seconds: -2}` |
| `10.0Newtons` | `{Kilograms: 1, Meters: 1, Seconds: -2}` |
| `ratio` (Hz/Hz) | `{}` (dimensionless) |

**Rules:**
- Multiplication: add exponents — `{A: 1} * {B: 1} = {A: 1, B: 1}`
- Division: subtract exponents — `{A: 1} / {A: 1} = {}` (cancels)
- Exponentiation: multiply exponents — `{A: 1} ^ 2 = {A: 2}`
- Addition/subtraction: exponent maps must be identical — `{A: 1} + {A: 1}` OK, `{A: 1} + {B: 1}` ERROR
- When all exponents are zero, the result is dimensionless

### Named Unit Matching

After computing a derived exponent map, the compiler checks if it matches any declared unit. If so, it uses the name for display and type annotations:

```
let v = 100.0Meters / 9.58Seconds
// exponent map: {Meters: 1, Seconds: -1}
// matches declaration: unit MetersPerSecond = Meters / Seconds
// displayed type: MetersPerSecond
```

If no named unit matches, the compiler displays the raw combination:

```
let weird = 100.0Meters * 50.0Kilograms
// exponent map: {Meters: 1, Kilograms: 1}
// no matching declaration
// displayed type: Meters·Kilograms
```

### Printing Derived Units

Derived unit values print with their resolved unit name when one exists:

```
print(speed)             // 10.44MetersPerSecond
print(rate)              // 48000.0SampleRate
```

When no named unit matches, print the derived form:

```
print(weird)             // 5000.0Meters·Kilograms
```

### A Music/Audio Example

```
unit Hz = Float
unit Seconds = Float
unit Samples = Int, Float
unit Beats = Ratio, Float

unit SampleRate = Samples / Seconds
unit Tempo = Beats / Seconds

let sr = 48000.0 Samples / 1.0Seconds       // SampleRate
let bpm = 120.0Beats / 60.0Seconds           // Tempo

// convert a beat duration to seconds
let beat_dur = 1.0Beats
let real_dur = beat_dur / bpm                 // Seconds
// {Beats: 1} / {Beats: 1, Seconds: -1} = {Seconds: 1} ✓

// convert seconds to samples
let sample_count = sr * real_dur              // Samples
// {Samples: 1, Seconds: -1} * {Seconds: 1} = {Samples: 1} ✓

// frequency to period
let freq = 440.0Hz
let period = 1.0Seconds / freq               // Seconds/Hz
// if Hz is declared as a base unit, this is {Seconds: 1, Hz: -1}
// if Hz were declared as unit Hz = Cycles / Seconds, it would cancel differently
```

---

## Implementation Roadmap

### Level 0 (v0.1): Simple Named Units

- Unit declarations with concrete base type lists
- Literal suffix syntax
- Auto-generated same-unit arithmetic and scalar operations
- Dimensionless arithmetic with same-representation values
- Unit-aware printing
- Cross-unit errors

**Internal representation:** Even at Level 0, represent units internally as exponent maps (`{Hz: 1}` rather than string tag `"Hz"`). This costs nothing extra and makes the upgrade to Level 2 seamless — you're just lifting restrictions, not changing data structures.

### Level 1 (v0.2): Unit-Polymorphic Functions

- Unit variables in generic parameter lists: `fun double[U : Float](x: U): U`
- Unit variable unification during type checking
- Representation constraints on unit variables

**Prerequisite:** Basic generic functions must work first.

### Level 2 (v0.3+): Derived Units

- `*`, `/`, `^` in unit declarations
- Exponent map algebra during type checking
- Named unit matching for derived exponent maps
- Derived unit display in printing and error messages

**Prerequisite:** Level 1 (unit variables are needed for full derived unit inference in generic code).

### Architecture Decisions to Make Now

**Exponent map from day one:** Even in Level 0, store units as `HashMap<String, i32>` internally. A simple unit like Hz is `{"Hz": 1}`. Dimensionless is `{}`. This representation naturally supports all three levels without refactoring.

**Unit registry:** The compiler/interpreter maintains a registry of declared units. Each entry stores the unit name, the allowed base representations, and the exponent map (at Level 0, always a single entry; at Level 2, potentially complex).

**Separation of concerns:** Unit checking is a layer ON TOP of regular type checking. First verify that the representations are compatible (Float + Float, not Float + Int), then verify that the unit exponent maps are compatible. This keeps the unit logic isolated and testable.

---

## Standard Library Units

### Music (Always Available)

```
unit Hz = Float, Int              // frequency
unit Cents = Float                // logarithmic pitch interval
unit Beats = Ratio, Int, Float    // musical duration
unit Samples = Int                // discrete audio samples
```

### Extended Music (import "std/units/music")

```
unit BPM = Float, Int
unit Decibels = Float
unit Velocity = Int               // MIDI velocity 0-127
unit Semitones = Int, Float
```

### Physics (import "std/units/physics")

```
unit Meters = Float
unit Seconds = Float
unit Kilograms = Float
unit MetersPerSecond = Meters / Seconds
unit Acceleration = Meters / (Seconds ^ 2)
unit Newtons = Kilograms * Meters / (Seconds ^ 2)
unit Pascals = Newtons / (Meters ^ 2)
unit Joules = Newtons * Meters
unit Watts = Joules / Seconds
```

### Audio DSP (import "std/units/dsp")

```
unit SampleRate = Samples / Seconds
unit Tempo = Beats / Seconds
unit Amplitude = Float            // dimensionless but semantically tagged
unit Phase = Float                // radians
```

---

## Interaction with Other Features

### UFCS

```
fun octave_above(f: Hz): Hz = f * 2.0

440.0Hz.octave_above()    // 880.0Hz
```

### Pattern Matching

```
fun describe_freq(f: Hz): String = match f {
    f when f < 20.0Hz => "infrasonic",
    f when f < 20000.0Hz => "audible",
    _ => "ultrasonic",
}
```

### Generators

```
let harmonics = gen n in 1..16 {
    440.0Hz * Float(n)
}
// type: Seq[Hz]
```

### Type Annotations

```
fun midi_to_freq(note: Int): Hz {
    Hz(440.0 * pow(2.0, Float(note - 69) / 12.0))
}

let freqs: List[Hz] = [440.0Hz, 880.0Hz, 1760.0Hz]
```

---

## Open Questions

- **Should derived unit declarations require all operand units to share a representation?** e.g., `unit Tempo = Beats / Seconds` — Beats supports Ratio and Float, Seconds supports Float. The derived unit would only work for the intersection (Float). Is this automatically inferred or does the user specify it?

- **Literal suffixes for derived units:** Can you write `10.0MetersPerSecond` as a literal, or only construct derived values through arithmetic (`10.0Meters / 1.0Seconds`)? Allowing literal suffixes for derived units is convenient but means the parser needs to recognize multi-word unit names.

- **Interaction with the `i` suffix:** Complex unit values (`Complex[Float] Hz`) are blocked unless the declaration includes Complex. Is this the right default? Some DSP work uses complex-valued frequencies (analytic signals). Could be opt-in: `unit Hz = Float, Int, Complex[Float]`.

- **Unit aliases:** Should there be a way to create a unit alias without creating a new base unit? e.g., `unit Frequency = Hz` — just another name for the same exponent map. Useful for readability but could cause confusion.

---

## Cross-RFC Discrepancies and Notes

This section documents known differences between this RFC and other documents. **This RFC (0006) is authoritative for unit type behavior.** Other documents should be updated to match.

### vs. RFC-0005 (Numeric Literals)

1. **Suffix casing.** RFC-0005 proposes lowercase suffixes (`440.0hz`, `701.955c`) with a scanner that splits `<number><alpha>` and auto-lowercases the type name. This RFC uses PascalCase suffixes (`440.0Hz`, `700.0Cents`) matching the declared type name exactly. **This RFC wins.** The spec (§2.4, §6.3) currently uses lowercase `hz`/`c` and will need updating.

2. **Suffix resolution fallback.** RFC-0005 describes a two-step resolution: try unit type match, then fall back to function call. This RFC treats suffixes as strict unit-name matching with no function fallback. Non-unit suffixes like `r` (for `Ratio`) should be handled separately — either as a dedicated scanner rule or as a UFCS function, not through the unit suffix mechanism.

3. **The `r` suffix.** RFC-0005 proposes `fun r(x: Int): Ratio = Ratio(x, 1)` as a stdlib function. Since `Ratio` is a core numeric type (not a `unit` declaration), the `r` suffix is outside the scope of this RFC's unit suffix system. It should remain a standalone UFCS function as RFC-0005 proposes.

4. **Dimensionless arithmetic.** RFC-0005's arithmetic table says `Hz + Float` is an **ERROR** ("Can't mix branded and unbranded"). This RFC says `440.0Hz + 20.0` is **OK** when representations match. **This RFC wins.** Allowing same-representation dimensionless arithmetic is more ergonomic for scalar operations (scaling, offsetting).

5. **`op_` convention.** RFC-0005 defines the `op_` naming convention for user-defined operator overloading. This RFC's auto-generated arithmetic implicitly uses that mechanism. The `op_` convention from RFC-0005 remains the canonical definition for operator overloading.

6. **`Int / Int → Ratio` interaction.** RFC-0005 establishes that `Int / Int → Ratio` always. For unit-tagged integers (e.g., `4Beats / 2Beats` where both are Int-backed), same-unit division produces a dimensionless result. The representation follows the core rule: `Int / Int → Ratio`, so the result is a dimensionless `Ratio`, not an `Int`.

7. **Multi-representation declarations.** RFC-0005's examples only show single base types (`unit Hz = Float`). This RFC extends declarations to multiple base types (`unit Hz = Float, Int`). This is an enrichment, not a conflict.

8. **EDO and the `\` operator.** RFC-0005 drops `\` as an infix operator, replacing EDO with `edo(7, 12)` / `7.edo(12)`. The `Edo` type is a product type carrying a division count at runtime, not a `unit` type (it has structural data beyond a single number). This RFC does not govern `Edo`.

### vs. Existing Spec

9. **§2.4 (Numeric Literals):** Currently describes `hz` and `c` as lowercase postfix on float literals, handled as UFCS calls. Needs updating to PascalCase unit suffixes as defined here.

10. **§3.2 (Music Types):** Lists `Hz` and `Cents` as built-in music types. Under this RFC they become stdlib `unit` declarations, not language primitives.

11. **§6.3 (Parsing Notes):** Describes `hz` and `c` as UFCS member access. Needs updating to describe unit suffix parsing per this RFC.

12. **§6.1 (EBNF Grammar):** No `unit` production exists yet. The grammar needs a new declaration form:
    ```
    unit_decl = "unit" IDENT "=" type_list
              | "unit" IDENT "=" unit_expr ;   // Level 2: derived units
    ```

### vs. RFC-0005 Open Questions Carried Forward

13. **`distinct` keyword.** RFC-0005 asks whether a `distinct` keyword should exist alongside `unit` for non-numeric branding (e.g., `type SQL = distinct String`). This remains an open question — `unit` only covers numeric types.

14. **`i` suffix ordering.** Neither RFC specifies what happens with `440.0Hzi` or `440.0iHz`. Since `i` changes mathematical classification and unit names label measurement, they are mutually exclusive on the same literal. The parser should reject any literal with both a unit suffix and `i`.
