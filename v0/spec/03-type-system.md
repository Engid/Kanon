# 3. Type System

> **Created:** 2026-04-05  
> **Last Updated:** 2026-04-06  
> **Changelog:**
> - 2026-04-05 — Initial draft
> - 2026-04-06 — Added metadata headers

## 3.1 Numeric Tower

Clef's numeric tower follows the Racket model: **exact by default, inexactness taints.**

```
        Complex[Ratio]    Complex[Float]
             │                  │
           Ratio              Float
             │
            Int
```

**Exactness rules:**
- `Int` op `Int` → `Int` (for `+`, `-`, `*`) or `Ratio` (for `/`)
- `Ratio` op `Ratio` → `Ratio`
- Anything op `Float` → `Float` (inexactness taints)
- `Complex` follows the same tainting rule for its components

**Auto-simplification:** Rationals are reduced to lowest terms (via GCD) on every arithmetic operation. `6/4` is stored as `3/2`. Numerator and denominator are arbitrary-precision integers (bignums).

## 3.2 Music Types

```
Interval    // abstract: any pitch interval (Ratio, Edo, or Cents)
Ratio       // exact rational interval (just intonation)
Edo         // equal-tempered step (carries division count at runtime)
Cents       // float-valued interval in cents
Hz          // frequency (float)
```

Conversions between these types are explicit:

```
cents(3/2)           // Ratio → Cents: ≈ 701.955c
cents(7\12)          // Edo → Cents: 700.0c
ratio(7\12)          // Edo → Ratio: nearest rational approximation
freq(7\12, 440.0hz)  // Edo → Hz: frequency relative to reference
float(3/2)           // Ratio → Float: 1.5
```

**Unit safety:** Arithmetic between incompatible music types is a type error unless explicitly converted.

```
3/2 + 5/4            // OK: Ratio + Ratio → Ratio
7\12 + 4\12          // OK: Edo + Edo → Edo (same division)
3/2 + 7\12           // ERROR: cannot add Ratio and Edo directly
3/2 + ratio(7\12)    // OK: explicit conversion first
```

## 3.3 Type Declarations

Clef has three type declaration forms: **product types**, **sum types**, and **type aliases**.

### Product Types (`type`)

A `type` with a block body defines an immutable product type (record) with named fields.

```
type Note {
    pitch: Interval,
    duration: Ratio,
    velocity: number,
}

type Event {
    time: Ratio,
    pitch: Interval,
    duration: Ratio,
    velocity: number,
    instrument: string,
}

type Voice {
    events: Seq[Note],
    offset_beats: Ratio,
    tempo_factor: Ratio,
    instrument_name: string,
    volume_level: number,
}
```

**Construction** uses the type name with named fields:

```
let n = Note { pitch: 7\12, duration: 1/4, velocity: 80 }
```

**Field access** via dot notation:

```
print(n.pitch)       // 7\12
print(n.duration)    // 1/4
```

**Structural equality:** Two values of the same type are equal if all fields are equal.

```
Note { pitch: 7\12, duration: 1/4, velocity: 80 }
    == Note { pitch: 7\12, duration: 1/4, velocity: 80 }   // true
```

**Copy-with-modification** via `.with()`:

```
let n2 = n.with(pitch: 9\12)
// n2 == Note { pitch: 9\12, duration: 1/4, velocity: 80 }
// n is unchanged
```

All product types are **immutable**. There are no mutable fields. Values have **value semantics** — conceptually, assignment copies the value (the implementation may use structural sharing or copy-on-write under the hood).

### Sum Types (`enum`)

An `enum` defines a tagged union (discriminated union). Each variant can optionally carry named data fields.

```
enum Music {
    Note(pitch: Interval, duration: Ratio, velocity: number),
    Rest(duration: Ratio),
    Seq(a: Music, b: Music),
    Par(a: Music, b: Music),
    Modify(control: Control, m: Music),
}

enum Control {
    Tempo(factor: Ratio),
    Transpose(interval: Interval),
    Instrument(name: string),
    Volume(level: number),
    Custom(name: string, value: any),
}

// variants without data
enum Mode {
    Major,
    Minor,
    Dorian,
    Phrygian,
    Lydian,
    Mixolydian,
    Aeolian,
    Locrian,
}
```

**Construction** — variants are constructed with the enum name and variant name. Fields can be positional or named:

```
// positional
let m = Music.Note(7\12, 1/4, 80)
let r = Music.Rest(1/1)
let mode = Mode.Dorian

// named (optional, for clarity)
let m = Music.Note(pitch: 7\12, duration: 1/4, velocity: 80)
```

**Destructuring** via `match` (see §4.5):

```
match m {
    Music.Note(p, d, v) => "note at ${p}",
    Music.Rest(d) => "rest for ${d}",
    Music.Seq(a, b) => "sequence",
    _ => "other"
}
```

When matching within a context where the enum type is known (e.g., a function taking `Music`), the enum name prefix can be omitted:

```
fun describe(m: Music): string = match m {
    Note(p, d, v) => "note at ${p}",
    Rest(d) => "rest for ${d}",
    Seq(a, b) => "sequence of ${describe(a)} and ${describe(b)}",
    Par(a, b) => "parallel",
    Modify(ctrl, inner) => "modified ${describe(inner)}",
}
```

Enum variants with fields are conceptually **inline record types** — the field names serve as documentation and enable named construction, but the variant itself is not a standalone type.

### Type Aliases (`type` with `=`)

A `type` declaration with `=` creates a type alias — a new name for an existing type.

```
type Scale = list[Interval]
type Chord = list[Interval]
type NoteSeq = Seq[Note]
type Pitch = Interval
```

The parser distinguishes aliases from product types by the presence of `=`: `type Name = ...` is an alias; `type Name { ... }` is a product type.

### Generic Types

Square brackets for type parameterization (Scala/Go convention).

```
Seq[Note]
list[Interval]
Map[string, Ratio]
```

Generic type parameters on functions:

```
fun first[T](seq: Seq[T]): T = seq.take(1).collect[0]

fun map[T, U](seq: Seq[T], f: (T) => U): Seq[U] { ... }
```

## 3.4 The Music Algebraic Type

The core structural type for composition, inspired by Hudak's Haskore/Euterpea:

```
enum Music {
    Note(pitch: Interval, duration: Ratio, velocity: number),
    Rest(duration: Ratio),
    Seq(a: Music, b: Music),
    Par(a: Music, b: Music),
    Modify(control: Control, m: Music),
}

enum Control {
    Tempo(factor: Ratio),
    Transpose(interval: Interval),
    Instrument(name: string),
    Volume(level: number),
    Custom(name: string, value: any),
}
```

`Music` is a **recursive algebraic type**. A `Music` value is a tree: leaves are `Note` and `Rest`, internal nodes are `Seq` (sequential), `Par` (parallel), and `Modify` (transformation wrapper).

Sequential and parallel composition are exposed as library functions, chainable via UFCS:

```
a.seq(b)          // Music.Seq(a, b)
a.par(b)          // Music.Par(a, b)
```

This means a phrase like:

```
note(0\12, 1/4).seq(note(4\12, 1/4)).seq(note(7\12, 1/2))
```

Produces the tree:

```
        Seq
       /   \
     Seq   Note(7\12, 1/2)
    /   \
Note    Note
(0\12)  (4\12)
```

Operator sugar (e.g., `++` for seq, `&` for par) may be added in a future version.

## 3.5 Algebraic Laws

The following equational properties hold for well-formed `Music` values. These are not compiler-enforced but are guaranteed by the standard library implementation and can be verified with property-based testing.

```
// seq is associative with rest(0) as identity
a.seq(b).seq(c)    ==  a.seq(b.seq(c))
a.seq(rest(0/1))   ==  a
rest(0/1).seq(a)   ==  a

// par is associative and commutative
a.par(b).par(c)    ==  a.par(b.par(c))
a.par(b)           ==  b.par(a)

// transpose distributes over seq and par
a.seq(b).transpose(i)  ==  a.transpose(i).seq(b.transpose(i))
a.par(b).transpose(i)  ==  a.transpose(i).par(b.transpose(i))

// transpose composes additively
m.transpose(a).transpose(b)  ==  m.transpose(a + b)

// invert is an involution
m.invert(x).invert(x)  ==  m

// retrograde is an involution
m.retrograde.retrograde  ==  m

// retrograde reverses sequential composition
a.seq(b).retrograde  ==  b.retrograde.seq(a.retrograde)
```

These laws enable equational reasoning about music. For example, you can prove that transposing a whole piece is the same as transposing each part individually, or that reversing twice is a no-op.
