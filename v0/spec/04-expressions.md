# 4. Expressions and Statements

> **Created:** 2026-04-05  
> **Last Updated:** 2026-04-06  
> **Changelog:**
> - 2026-04-05 — Initial draft
> - 2026-04-06 — Added metadata headers

## 4.1 Variable Declarations

```
let x = 42                // immutable binding
let x: Ratio = 3/2        // with type annotation
mut count = 0              // mutable binding
```

Immutable bindings (`let`) cannot be reassigned. Mutable bindings (`mut`) can.

```
let a = 1
a = 2       // ERROR: cannot reassign immutable binding

mut b = 1
b = 2       // OK
```

## 4.2 Function Declarations

The `fun` keyword declares a named function. Single-expression bodies use `=`. Multi-statement bodies use braces. The last expression in a braced body is the return value.

```
// single-expression body
fun fifth(root: Ratio): Ratio = root * 3/2

// multi-statement body
fun quantize(pitch: Interval, scale: list[Interval]): Interval {
    let dists = scale.map(s => abs(cents(s) - cents(pitch)))
    scale(dists.index_of_min)
}

// generic function
fun first[T](seq: Seq[T]): T = seq.take(1).collect(0)

// function with default parameter
fun note(pitch: Interval, duration: Ratio, velocity: number = 80): Music =
    Music.Note(pitch, duration, velocity)
```

## 4.3 Lambda Expressions

Fat-arrow syntax (JS/C# style). Single-expression lambdas need no braces. Multi-statement lambdas use braces.

```
// single expression
let double = (x: Ratio) => x * 2
notes.map(n => n.pitch + 3\19)

// multi-statement
let process = (n: Note) => {
    let p = quantize(n.pitch, scale)
    note(p, n.duration)
}

// type-annotated
let f: (Ratio) => Ratio = (x) => x * 3/2

// no parameters
let greet = () => print("hello")
```

When a lambda has a single parameter, the parentheses around it are optional:

```
notes.map(n => n.pitch)       // OK
notes.map((n) => n.pitch)     // also OK
notes.filter(n => n.velocity > 60)
```

## 4.4 Control Flow

Braces are **mandatory** for all control flow bodies.

### If/Else (Expression)

`if` is an expression — it returns a value.

```
let vel = if step == chord[0] { 100 } else { 70 }

if pitch > 1200.0c {
    print("above octave")
} else if pitch > 700.0c {
    print("above fifth")
} else {
    print("below fifth")
}
```

### For Loop (Imperative)

`for` is always an imperative loop. It does not produce a value. For lazy sequence generation, use `gen` (§4.6).

```
for n in melody {
    print(n.pitch)
}

for i in 0..12 {
    print(i * 100.0c)
}
```

### While Loop

```
mut i = 0
while i < 10 {
    print(i)
    i = i + 1
}
```

## 4.5 Pattern Matching

Match arms are separated by **commas**. The last arm's trailing comma is optional.

```
fun describe(m: Music): string = match m {
    Note(p, d, v) => "note: ${p}",
    Rest(d) => "rest: ${d}",
    Seq(a, b) => "sequence",
    Par(a, b) => "parallel",
    Modify(Transpose(i), m) => "transposed by ${i}",
    _ => "other"
}
```

### Guards

```
match ratio {
    r when r == 3/2 => "perfect fifth",
    r when r == 5/4 => "major third",
    r when r > 1 and r < 2 => "within octave",
    _ => "other interval"
}
```

### Block Bodies

Match arms can have block bodies for multi-statement logic:

```
match event {
    Note(p, d, v) => {
        let adjusted = quantize(p, scale)
        note(adjusted, d, v)
    },
    Rest(d) => rest(d),
    _ => rest(0/1)
}
```

### Nested Patterns

Patterns can be nested to match inner structure:

```
match m {
    Modify(Transpose(i), Note(p, d, v)) => {
        // matched a transposed note specifically
        note(p + i, d, v)
    },
    Modify(Tempo(f), inner) => inner.scale_tempo(f),
    _ => m
}
```

## 4.6 Generators

The `gen` keyword produces a lazy `Seq[T]`. Generators use **implicit yield** for single expressions and **explicit `yield`** for multi-statement bodies.

### Simple Generators (Implicit Yield)

When the body is a single expression, it is implicitly yielded. Braces are optional for one-liners.

```
// one-liner, no braces
let overtones = gen n in 1.. n * fundamental

// one-liner with braces
let fifths = gen n in 0..12 { n * 7\12 }
```

### Filtered Generators

The `where` clause filters which iterations produce values.

```
let odd_harmonics = gen n in 1.. where n % 2 != 0 {
    n * fundamental
}
```

### Multi-Yield Generators

When the body contains explicit `yield` statements, multiple values can be emitted per iteration.

```
let arp_pairs = gen step in cycle(chord) {
    yield note(step, 1/8)
    yield note(step + 7\19, 1/16)
}
```

### Nested Generators

Generators can produce sub-sequences that are then flattened.

```
let phrases = gen c in cycle(chords) {
    gen p in c { note(p, 1/8) }.take(8)
}.flatten
```

### Distinction from `for` loops

`gen` **always** produces a lazy `Seq[T]`. `for` is **always** an imperative loop with side effects. They are syntactically and semantically distinct — there is no ambiguity.

| Construct | Produces a value? | Lazy? | Use case |
|-----------|------------------|-------|----------|
| `for x in seq { body }` | No | No | Side effects: printing, accumulating into `mut` vars |
| `gen x in seq expr` | Yes: `Seq[T]` | Yes | Constructing lazy sequences |

## 4.7 UFCS (Uniform Function Call Syntax)

Any function `f(x, y, z)` can be called as `x.f(y, z)`. The compiler rewrites dot-call syntax to a free function call, using the receiver as the first argument.

```
// these are identical:
map(notes, n => n.pitch)
notes.map(n => n.pitch)

// chaining via UFCS:
notes
    .map(n => n.pitch + 3\19)
    .filter(p => cents(p) < 1200.0c)
    .take(16)
```

User-defined functions are immediately chainable — no extension method declarations needed:

```
fun quantize_to(seq: Seq[Note], scale: list[Interval]): Seq[Note] {
    seq.map(n => Note { ...n, pitch: nearest(scale, n.pitch) })
}

// works as a method via UFCS:
melody.quantize_to(major_scale).take(32)
```

**Resolution order:** When `x.f(args)` is encountered:
1. First, look for a field `f` on the type of `x` (for product types).
2. Then, look for a free function `f` in scope where the first parameter matches the type of `x`.
3. If neither is found, it's a compile error.

This means product type fields take precedence over UFCS functions of the same name.

## 4.8 Pipe Operator

The pipe operator `|>` is available as an alternative to UFCS chaining. `x |> f(y)` is equivalent to `f(x, y)`.

```
notes
    |> map(n => n.pitch + 3\19)
    |> filter(p => cents(p) < 1200.0c)
    |> take(16)
```

UFCS dot-chaining is preferred in idiomatic Kanon, but the pipe operator exists for cases where it reads better (e.g., piping into a function that doesn't take a collection as its first argument).
