# Tone — A Language for Microtonal Composition

**Version 0.1 — Draft Specification**

*A domain-specific language for composing microtonal music with first-class rational numbers, lazy generators, algebraic music types, and C-family syntax.*

---

## 1. Design Philosophy

Tone is a garbage-collected, interpreted scripting language designed to be embedded in a host audio engine (analogous to how Lua, sclang, or Python are used in audio systems). The host handles real-time sample-level rendering; Tone handles composition, pattern generation, and structural manipulation.

### Core Principles

- **Numerical precision by default.** Rational numbers are exact. Floating-point is opt-in. Just intonation intervals are ratios, not approximations.
- **Composability.** Everything composes: functions via UFCS, sequences via generators, musical structures via algebraic operators. User-defined operations are indistinguishable from built-ins (Steele's growability principle).
- **Familiarity.** C-family syntax with TypeScript-style type annotations. Fat-arrow lambdas. Curly braces. A JS/C# developer should be able to read Tone code on day one.
- **Music-native types.** EDO steps, cents, frequencies, and intervals are first-class with literal syntax and unit safety.

### Architecture

```
┌─────────────────────────────┐
│  Tone Source Code            │
│  (.tone files)               │
└────────────┬────────────────┘
             │ parse
             ▼
┌─────────────────────────────┐
│  AST / Bytecode              │
│  (Pratt parser, bytecode VM) │
└────────────┬────────────────┘
             │ evaluate
             ▼
┌─────────────────────────────┐
│  Music Trees + Event Lists   │
│  (realize → timed events)    │
└────────────┬────────────────┘
             │ FFI
             ▼
┌─────────────────────────────┐
│  Host Audio Engine           │
│  (Rust/C/C++ — rendering,    │
│   synthesis, MIDI output)    │
└─────────────────────────────┘
```

---

## 2. Lexical Grammar

### 2.1 Comments

```
// single-line comment

/* multi-line
   comment */
```

### 2.2 Identifiers

Identifiers start with a letter or underscore, followed by letters, digits, or underscores. Type names are capitalized by convention. Variable and function names are lowercase or snake_case.

```
my_var   _temp   Note   Seq   EDO19   x2
```

### 2.3 Keywords

```
func    let     mut     type    match   case
if      else    for     in      gen     yield
return  true    false   nil     import  as
where   and     or      not
```

### 2.4 Numeric Literals

#### Integers
```
42        // decimal
0xFF      // hexadecimal
0b1010    // binary
0o77      // octal
1_000_000 // underscores for readability
```

#### Rationals (exact)
The `/` operator between two integer literals produces an exact rational value. This is a literal form, not a division operation.
```
3/2       // Ratio: exactly three-halves
5/4       // Ratio: the just major third
7/1       // Ratio: integer promoted to rational
```

#### Floats (inexact)
Any numeric literal with a decimal point is a float. Floats use IEEE 754 double precision.
```
3.14      // Float
1.0       // Float (not integer, not rational)
6.02e23   // Float with exponent
```

#### Complex
Postfix `i` on any numeric literal produces the imaginary component. Complex numbers have either rational or float components depending on the literal type.
```
2i          // Complex<Int>: 0 + 2i
3/2 + 1/2i  // Complex<Ratio>: (3/2) + (1/2)i
1.5 + 0.5i  // Complex<Float>: 1.5 + 0.5i
```

#### EDO Step Literals
The `\` operator between two integer literals produces an EDO (Equal Division of the Octave) step. This uses the established xenharmonic community convention.
```
7\12      // step 7 of 12-EDO (≈ perfect fifth)
1\12      // step 1 of 12-EDO (semitone)
6\19      // step 6 of 19-EDO
9\31      // step 9 of 31-EDO (neutral third)
0\53      // unison in 53-EDO
```

The type of an EDO literal is `Edo`, carrying its division count at runtime.

#### Unit-Typed Literals
```
440.0hz   // Frequency in Hertz
100.0c    // Interval in cents
```

### 2.5 String Literals

Double-quoted strings with interpolation.
```
"hello"
"pitch: ${n.pitch}"
"line 1\nline 2"
```

### 2.6 Operators

```
// arithmetic
+  -  *  /  %

// comparison
==  !=  <  <=  >  >=

// logical
and  or  not

// assignment
=

// pipe (available, but UFCS is preferred)
|>

// range
..     // exclusive end
..=    // inclusive end
```

---

## 3. Type System

### 3.1 Numeric Tower

Tone's numeric tower follows the Racket model: **exact by default, inexactness taints.**

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

### 3.2 Music Types

```
Interval    // abstract: any pitch interval (Ratio, Edo, or Cents)
Ratio       // exact rational interval (just intonation)
Edo         // equal-tempered step (carries division count)
Cents       // float-valued interval in cents
Hz          // frequency (float)
```

Conversions between these types are explicit:
```
cents(3/2)         // Ratio → Cents: ≈ 701.955c
cents(7\12)        // Edo → Cents: 700.0c
ratio(7\12)        // Edo → Ratio: approximation
freq(7\12, 440.0hz) // Edo → Hz: relative to reference
```

**Unit safety:** Arithmetic between incompatible music types is a type error unless explicitly converted.
```
3/2 + 5/4          // OK: Ratio + Ratio → Ratio
7\12 + 4\12        // OK: Edo + Edo → Edo (same division)
3/2 + 7\12         // ERROR: cannot add Ratio and Edo directly
3/2 + ratio(7\12)  // OK: Ratio + Ratio → Ratio
```

### 3.3 Generic Types

Square brackets for type parameterization (Scala/Go convention).
```
Seq[Note]
list[Interval]
Map[string, Ratio]
```

### 3.4 The Music Algebraic Type

The core structural type for composition, inspired by Hudak's Haskore/Euterpea:
```
type Music {
    Note(pitch: Interval, duration: Ratio, velocity: number)
    Rest(duration: Ratio)
    Seq(a: Music, b: Music)       // sequential: a then b
    Par(a: Music, b: Music)       // parallel: a and b simultaneously
    Modify(control: Control, m: Music)
}

type Control {
    Tempo(factor: Ratio)
    Transpose(interval: Interval)
    Instrument(name: string)
    Volume(level: number)
    Custom(name: string, value: any)
}
```

Sequential and parallel composition are exposed as library functions, chainable via UFCS:
```
a.seq(b)          // sequential composition
a.par(b)          // parallel composition
```

Operator sugar may be added in a future version (e.g., `++` for seq, `&` for par).

### 3.5 Algebraic Laws

The following equational properties hold for well-formed `Music` values. These are not compiler-enforced but are guaranteed by the standard library implementation and can be verified with property-based testing.

```
// seq is associative with Rest(0) as identity
a.seq(b).seq(c)  ==  a.seq(b.seq(c))
a.seq(rest(0))   ==  a
rest(0).seq(a)    ==  a

// par is associative and commutative
a.par(b).par(c)  ==  a.par(b.par(c))
a.par(b)         ==  b.par(a)

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

---

## 4. Expressions and Statements

### 4.1 Variable Declarations

```
let x = 42              // immutable binding
let x: Ratio = 3/2      // with type annotation
mut count = 0            // mutable binding
```

### 4.2 Function Declarations

Single-expression functions use `=`. Multi-statement functions use braces.
```
// single-expression
func fifth(root: Ratio): Ratio = root * 3/2

// multi-statement
func quantize(pitch: Interval, scale: list[Interval]): Interval {
    let dists = scale.map(s => abs(cents(s) - cents(pitch)))
    scale[dists.index_of_min]
}

// generic function
func first[T](seq: Seq[T]): T = seq.take(1).collect[0]
```

### 4.3 Lambda Expressions

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

### 4.4 Control Flow

Braces are mandatory for all control flow bodies.

```
// if/else (expression — returns a value)
let vel = if step == chord[0] { 100 } else { 70 }

// multi-line
if pitch > 1200.0c {
    print("above octave")
} else if pitch > 700.0c {
    print("above fifth")
} else {
    print("below fifth")
}

// for loop (imperative — no value produced)
for n in melody {
    print(n.pitch)
}

// for with range
for i in 0..12 {
    print(i * 100.0c)
}

// while loop
mut i = 0
while i < 10 {
    print(i)
    i = i + 1
}
```

### 4.5 Pattern Matching

```
match interval {
    Note(p, d, v) => "note: ${p}"
    Rest(d) => "rest: ${d}"
    Seq(a, b) => "sequence"
    Par(a, b) => "parallel"
    Modify(Transpose(i), m) => "transposed by ${i}"
    _ => "other"
}

// with guards
match ratio {
    r when r == 3/2 => "perfect fifth"
    r when r == 5/4 => "major third"
    r when r > 1 and r < 2 => "within octave"
    _ => "other interval"
}
```

### 4.6 Generators

The `gen` keyword produces a lazy `Seq[T]`. Generators use implicit yield for single expressions, explicit `yield` for multi-statement bodies.

```
// single expression — implicit yield, braces optional
let overtones = gen n in 1.. n * fundamental

// single expression with braces
let fifths = gen n in 0..12 { n * 7\12 }

// with filter clause
let odd_harmonics = gen n in 1.. where n % 2 != 0 {
    n * fundamental
}

// multi-yield — explicit yield required
let arp_pairs = gen step in cycle(chord) {
    yield note(step, 1/8)
    yield note(step + 7\19, 1/16)
}

// nested generators
let phrases = gen c in cycle(chords) {
    gen p in c { note(p, 1/8) }.take(8)
}
```

**Distinction from `for` loops:** `gen` always produces a lazy `Seq[T]`. `for` is always an imperative loop with side effects. They are syntactically and semantically distinct.

### 4.7 UFCS (Uniform Function Call Syntax)

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

// user-defined functions are immediately chainable:
func quantize_to(seq: Seq[Note], scale: list[Interval]): Seq[Note] {
    seq.map(n => note(nearest(scale, n.pitch), n.duration))
}

// works as a method:
melody.quantize_to(major_scale).take(32)
```

**Resolution order:** When `x.f(args)` is encountered, the compiler first looks for a method `f` defined on the type of `x`. If none is found, it looks for a free function `f` where the first parameter matches the type of `x`. This means true methods (defined in `type` blocks) take precedence over UFCS.

---

## 5. Standard Library

### 5.1 Sequence Operations

All sequence operations are free functions, chainable via UFCS.

```
// creation
range(start, end)           // finite range
repeat(value)               // infinite repetition of a value
cycle(list)                 // infinite cycling over a list
empty[T]()                  // empty sequence

// transformation
map[T, U](seq: Seq[T], f: (T) => U): Seq[U]
flat_map[T, U](seq: Seq[T], f: (T) => Seq[U]): Seq[U]
filter[T](seq: Seq[T], f: (T) => bool): Seq[T]
flatten[T](seq: Seq[Seq[T]]): Seq[T]
zip[T, U](a: Seq[T], b: Seq[U]): Seq[(T, U)]
scan[T, U](seq: Seq[T], init: U, f: (U, T) => U): Seq[U]
chunk[T](seq: Seq[T], size: number): Seq[list[T]]

// consumption
take[T](seq: Seq[T], n: number): Seq[T]
drop[T](seq: Seq[T], n: number): Seq[T]
take_while[T](seq: Seq[T], f: (T) => bool): Seq[T]
collect[T](seq: Seq[T]): list[T]      // forces evaluation
reduce[T](seq: Seq[T], f: (T, T) => T): T
fold[T, U](seq: Seq[T], init: U, f: (U, T) => U): U
find[T](seq: Seq[T], f: (T) => bool): T?

// reordering
reverse[T](seq: Seq[T]): Seq[T]       // requires finite seq
rotate[T](seq: Seq[T], n: number): Seq[T]
sort_by[T, U: Ord](seq: Seq[T], f: (T) => U): Seq[T]
```

### 5.2 List Operations

```
len[T](l: list[T]): number
push[T](l: list[T], item: T): list[T]
concat[T](a: list[T], b: list[T]): list[T]
slice[T](l: list[T], start: number, end: number): list[T]
index_of[T](l: list[T], item: T): number?
contains[T](l: list[T], item: T): bool
```

### 5.3 Math Functions

```
abs(x: number): number
max(a: number, b: number): number
min(a: number, b: number): number
floor(x: Float): Int
ceil(x: Float): Int
round(x: Float): Int
sqrt(x: Float): Float
pow(base: number, exp: number): number
log(x: Float): Float
log2(x: Float): Float
sin(x: Float): Float
cos(x: Float): Float

// rational-specific
numer(r: Ratio): Int          // numerator
denom(r: Ratio): Int          // denominator
simplify(r: Ratio): Ratio     // reduce to lowest terms (auto on construction)
float(r: Ratio): Float        // explicit inexact conversion

// integer division (free functions, chainable via UFCS)
div(a: Int, b: Int): Int      // truncating integer division
mod(a: Int, b: Int): Int      // remainder
```

### 5.4 Music Construction

```
// primitives
note(pitch: Interval, duration: Ratio): Music
note(pitch: Interval, duration: Ratio, velocity: number): Music
rest(duration: Ratio): Music

// composition (chainable via UFCS)
seq(a: Music, b: Music): Music         // sequential
par(a: Music, b: Music): Music         // parallel

// transformations (chainable via UFCS)
transpose(m: Music, interval: Interval): Music
invert(m: Music, axis: Interval): Music
retrograde(m: Music): Music
scale_tempo(m: Music, factor: Ratio): Music
set_instrument(m: Music, name: string): Music
set_volume(m: Music, level: number): Music

// queries
duration(m: Music): Ratio
pitches(m: Music): list[Interval]
```

### 5.5 Tuning and Scales

```
// EDO construction
edo(divisions: Int): list[Edo]                        // all steps of n-EDO
edo(divisions: Int, period: Ratio): list[Edo]         // non-octave EDOs

// conversion
cents(i: Interval): Cents
ratio(e: Edo): Ratio               // nearest rational approximation
ratio(c: Cents): Ratio             // nearest rational approximation
freq(i: Interval, ref: Hz): Hz     // to frequency given reference

// scale utilities
nearest(scale: list[Interval], pitch: Interval): Interval
mode(scale: list[Interval], degree: Int): list[Interval]

// just intonation
limit(n: Int): list[Ratio]         // all n-limit ratios within an octave
odd_limit(n: Int): list[Ratio]     // all n-odd-limit ratios
```

### 5.6 Mixing and Arrangement

`mix` is a library function that combines voices for parallel playback.

```
// voice wraps a Seq[Note] with playback metadata
func voice(seq: Seq[Note]): Voice

// voice modifiers (UFCS-chainable)
func offset(v: Voice, beats: Ratio): Voice
func tempo(v: Voice, factor: Ratio): Voice
func instrument(v: Voice, name: string): Voice
func volume(v: Voice, level: number): Voice

// mix combines voices
func mix(voices: list[Voice]): Music
```

Usage:
```
let piece = mix([
    voice(bass_line),
    voice(chords),
    voice(arp).offset(1/4),
    voice(drone).tempo(5/4).volume(0.6),
])
```

### 5.7 Realization and Rendering

```
// Music → event stream
func realize(m: Music): Seq[Event]

// Event is a concrete timed note
type Event {
    time: Ratio           // beat position
    pitch: Interval
    duration: Ratio
    velocity: number
    instrument: string
}

// rendering (host FFI)
func render(events: Seq[Event], bpm: number, ref: Hz, sample_rate: Int): Audio
func export(audio: Audio, path: string): nil
func to_midi(events: Seq[Event], path: string): nil

// conversion between structural and generative layers
func to_music(seq: Seq[Note]): Music    // Seq[Note] → Music (sequential chain)
func take_beats(seq: Seq[Note], beats: Ratio): Seq[Note]
```

---

## 6. Grammar (EBNF)

A simplified formal grammar. The actual parser would use recursive descent with Pratt parsing for expressions.

```ebnf
program        = { declaration } ;

declaration    = func_decl
               | type_decl
               | let_decl
               | statement ;

(* --- Declarations --- *)

func_decl      = "func" IDENT [ type_params ] "(" [ param_list ] ")" [ ":" type ]
                 ( "=" expression | block ) ;

type_decl      = "type" IDENT [ type_params ] "{" { variant } "}" ;
variant        = IDENT [ "(" param_list ")" ] ;

let_decl       = ( "let" | "mut" ) IDENT [ ":" type ] "=" expression ;

(* --- Statements --- *)

statement      = expression
               | for_loop
               | while_loop
               | return_stmt ;

for_loop       = "for" IDENT "in" expression block ;
while_loop     = "while" expression block ;
return_stmt    = "return" [ expression ] ;

block          = "{" { declaration } "}" ;

(* --- Expressions (ordered by ascending precedence in Pratt parser) --- *)

expression     = assignment ;

assignment     = IDENT "=" expression
               | pipe_expr ;

pipe_expr      = or_expr { "|>" or_expr } ;

or_expr        = and_expr { "or" and_expr } ;
and_expr       = equality { "and" equality } ;
equality       = comparison { ( "==" | "!=" ) comparison } ;
comparison     = addition { ( "<" | "<=" | ">" | ">=" ) addition } ;
addition       = multiplication { ( "+" | "-" ) multiplication } ;
multiplication = unary { ( "*" | "/" | "%" ) unary } ;
unary          = ( "-" | "not" ) unary | postfix ;
postfix        = primary { call | index | dot | imaginary } ;

call           = "(" [ arg_list ] ")" ;
index          = "[" expression "]" ;
dot            = "." IDENT [ call ] ;
imaginary      = "i" ;         (* postfix imaginary marker *)

primary        = NUMBER
               | RATIO         (* integer "/" integer *)
               | EDO           (* integer "\" integer *)
               | FLOAT
               | STRING
               | "true" | "false" | "nil"
               | IDENT
               | lambda
               | gen_expr
               | if_expr
               | match_expr
               | list_literal
               | "(" expression ")" ;

(* --- Lambda --- *)

lambda         = "(" [ param_list ] ")" "=>" ( expression | block ) ;

(* --- Generator --- *)

gen_expr       = "gen" IDENT "in" expression [ "where" expression ]
                 ( expression | block ) ;

(* --- If (expression) --- *)

if_expr        = "if" expression block [ "else" ( if_expr | block ) ] ;

(* --- Match --- *)

match_expr     = "match" expression "{" { match_arm } "}" ;
match_arm      = pattern [ "when" expression ] "=>" expression ;

pattern        = IDENT "(" [ pattern_list ] ")"   (* constructor *)
               | IDENT                             (* binding *)
               | "_"                               (* wildcard *)
               | literal ;                         (* literal *)

(* --- Types --- *)

type           = IDENT [ "[" type_list "]" ]       (* simple or generic *)
               | "(" [ type_list ] ")" "=>" type   (* function type *)
               | type "?" ;                        (* optional *)

type_params    = "[" IDENT { "," IDENT } "]" ;

(* --- Misc --- *)

param_list     = param { "," param } ;
param          = IDENT [ ":" type ] [ "=" expression ] ;
arg_list       = expression { "," expression } ;
list_literal   = "[" [ expression { "," expression } ] "]" ;
```

### 6.1 Pratt Parser Precedence Table

The expression parser uses a Pratt (top-down operator precedence) parser. Each token has a prefix and/or infix handler with an associated binding power.

| Precedence | Operators              | Associativity | Notes                              |
|------------|------------------------|---------------|------------------------------------|
| 1          | `=`                    | Right         | Assignment                         |
| 2          | `\|>`                  | Left          | Pipe                               |
| 3          | `or`                   | Left          | Logical OR                         |
| 4          | `and`                  | Left          | Logical AND                        |
| 5          | `==` `!=`              | Left          | Equality                           |
| 6          | `<` `<=` `>` `>=`     | Left          | Comparison                         |
| 7          | `+` `-`               | Left          | Addition                           |
| 8          | `*` `/` `%`           | Left          | Multiplication, rational literal   |
| 9          | `-` `not`             | Right (prefix)| Unary                              |
| 10         | `()` `[]` `.`         | Left          | Call, index, member access         |
| 11         | `i`                   | Left (postfix)| Imaginary suffix                   |
| 12         | `\`                   | Left          | EDO literal (integer \ integer)    |

**Special parsing rules:**
- `/` between two integer literals is parsed as a `Ratio` literal, not a division. The Pratt parser checks the types of left and right operands after parsing: if both are integer literals, emit a `Ratio` node; otherwise emit a `Div` node.
- `\` between two integer literals is parsed as an `Edo` literal. It is an error if either operand is not an integer literal.
- `i` as a postfix on a numeric literal produces an imaginary component. The Pratt parser checks that the preceding expression is a numeric literal.
- `hz` and `c` as postfix identifiers on float literals produce `Hz` and `Cents` values respectively. These are handled as UFCS calls to conversion functions.

---

## 7. Example Programs

### 7.1 Hello World — A Simple Melody

```
let scale = [0\12, 2\12, 4\12, 5\12, 7\12, 9\12, 11\12]

let melody = scale
    .map(p => note(p, 1/4))
    .reduce((a, b) => a.seq(b))

melody.realize
    .render(bpm: 120, ref: 262.0hz, sample_rate: 48000)
    .export("hello.wav")
```

### 7.2 Just Intonation Chord

```
let root = 1/1
let major_chord = [root, 5/4, 3/2]

let chord = major_chord
    .map(p => note(p, 1/1))
    .reduce((a, b) => a.par(b))

chord.realize
    .render(bpm: 60, ref: 220.0hz, sample_rate: 48000)
    .export("just_chord.wav")
```

### 7.3 Generative Arpeggio in 19-EDO

```
let edo19_major = [0\19, 3\19, 6\19, 8\19, 11\19, 14\19, 17\19]

let arp = gen p in cycle(edo19_major) note(p, 1/8)

let four_bars = arp.take(32).to_music

four_bars.realize
    .render(bpm: 140, ref: 262.0hz, sample_rate: 48000)
    .export("arp_19edo.wav")
```

### 7.4 Polymetric Composition in 31-EDO

```
let major = [0\31, 5\31, 10\31, 13\31, 18\31, 23\31, 28\31]

func arpeggiate(chord: list[Interval], dur: Ratio): Seq[Note] =
    gen p in cycle(chord) note(p, dur)

let chords = [
    [major[0], major[2], major[4]],
    [major[5], major[0], major[2]],
    [major[4], major[6], major[1]],
    [major[3], major[5], major[0]],
]

let bass = gen c in cycle(chords) note(c[0], 1/1)
let pads = gen c in cycle(chords) {
    c.map(p => note(p, 1/2)).reduce((a, b) => a.par(b))
}
let arp = gen c in cycle(chords) {
    arpeggiate(c, 1/8).take(8)
} .flatten

let piece = mix([
    voice(bass),
    voice(pads),
    voice(arp),
    voice(
        gen p in cycle(major.reverse) note(p + 18\31, 1/4)
    ).offset(1/4),
])

piece.realize
    .take_beats(32 * 4/1)
    .render(bpm: 120, ref: 262.0hz, sample_rate: 48000)
    .export("poly_31edo.wav")
```

### 7.5 Structural Composition with Transformations

```
let major = [0\31, 5\31, 10\31, 13\31, 18\31, 23\31, 28\31]

func phrase(steps: list[Interval], dur: Ratio): Music =
    steps.map(s => note(s, dur)).reduce((a, b) => a.seq(b))

func harmonize(m: Music, interval: Interval): Music =
    m.par(m.transpose(interval))

// build structure
let theme = phrase(major, 1/4)

let section_a = theme.seq(theme.transpose(13\31))
let section_b = theme.invert(0\31).seq(theme.retrograde)
let chorus = theme.harmonize(18\31)

let piece = section_a
    .seq(chorus)
    .seq(section_b)
    .seq(chorus)

// analyze
print("Duration: ${duration(piece)} beats")
print("Pitches used: ${pitches(piece).len}")

// render
piece.realize
    .render(bpm: 108, ref: 262.0hz, sample_rate: 48000)
    .export("structure.wav")
```

### 7.6 Bohlen-Pierce Scale (Non-Octave Tuning)

```
// Bohlen-Pierce: 13 equal divisions of the tritave (3:1)
let bp = edo(13, 3/1)

// A Bohlen-Pierce "triad" (approximating 3:5:7)
let bp_triad = [bp[0], bp[4], bp[6]]

let melody = gen step in cycle([0, 4, 6, 10, 6, 4]) {
    note(bp[step], 1/4)
}

let drone = gen _ in repeat(nil) {
    note(bp[0], 1/1)
}

let piece = mix([
    voice(melody),
    voice(drone).volume(0.4),
])

piece.realize
    .take_beats(16/1)
    .render(bpm: 72, ref: 262.0hz, sample_rate: 48000)
    .export("bohlen_pierce.wav")
```

### 7.7 Pattern Matching on Music Structure

```
// count all notes in a Music tree
func count_notes(m: Music): Int = match m {
    Note(_, _, _) => 1
    Rest(_) => 0
    Seq(a, b) => count_notes(a) + count_notes(b)
    Par(a, b) => count_notes(a) + count_notes(b)
    Modify(_, inner) => count_notes(inner)
}

// extract all pitches
func all_pitches(m: Music): list[Interval] = match m {
    Note(p, _, _) => [p]
    Rest(_) => []
    Seq(a, b) => all_pitches(a).concat(all_pitches(b))
    Par(a, b) => all_pitches(a).concat(all_pitches(b))
    Modify(Transpose(i), inner) => all_pitches(inner).map(p => p + i)
    Modify(_, inner) => all_pitches(inner)
}

// compute pitch-class histogram for analysis
func pc_histogram(m: Music, divisions: Int): list[Int] {
    mut hist = list.fill(divisions, 0)
    for p in all_pitches(m) {
        let pc = cents(p).div(1200.0 / float(divisions)).floor.mod(divisions)
        hist[pc] = hist[pc] + 1
    }
    hist
}

let piece = phrase(major, 1/4).harmonize(18\31)
print("Notes: ${count_notes(piece)}")
print("Pitch classes: ${pc_histogram(piece, 31)}")
```

### 7.8 DFT on Pitch-Class Sets (Complex Number Usage)

```
// Discrete Fourier Transform on a pitch-class set
// Uses complex numbers with rational components for exact computation
// (following David Lewin's generalized interval systems)

let pi = 3.14159265358979

func dft(pc_set: list[Int], n: Int): list[Complex[Float]] {
    gen k in 0..n {
        pc_set.map(p => {
            let angle = -2.0 * pi * float(k * p) / float(n)
            cos(angle) + sin(angle) * 1.0i
        }).reduce((a, b) => a + b)
    } .collect
}

// 12-TET major scale as pitch classes
let major_12 = [0, 2, 4, 5, 7, 9, 11]
let spectrum = dft(major_12, 12)

for k in 0..12 {
    let magnitude = sqrt(spectrum[k].real.pow(2) + spectrum[k].imag.pow(2))
    print("k=${k}: magnitude=${magnitude}")
}
```

---

## 8. Implementation Notes

### 8.1 Parsing Strategy

Use a hand-rolled recursive descent parser with Pratt parsing for expressions. The scanner produces tokens; the parser consumes them to build an AST.

**Key parsing challenges:**
- `/` as rational literal vs. division: check if both operands are integer literals after parsing a `*`/`/` precedence expression.
- `\` as EDO literal: high precedence, both operands must be integer literals.
- `i` as imaginary suffix: postfix on numeric literals only.
- `[]` disambiguation: type position → generic; value position → index; bare → list literal.
- `gen` vs `for`: `gen` always starts a generator expression; `for` always starts an imperative loop statement.

### 8.2 Runtime

- **Garbage collection:** Tracing GC (mark-and-sweep or generational). Music trees and generator closures are heap-allocated.
- **Rational arithmetic:** Arbitrary-precision integers (bignum) for numerator and denominator. Auto-simplify (GCD reduction) on every operation.
- **Lazy sequences:** Generators are implemented as coroutines or closure-based iterators. A `Seq[T]` is an opaque iterator object with a `.next()` method.
- **UFCS resolution:** At compile time (or during name resolution), `x.f(args)` is rewritten to `f(x, args)` if `f` is not a method on the type of `x`.

### 8.3 Host FFI

The boundary between Tone and the host audio engine is the `Event` type. `realize` walks the `Music` tree and produces a time-sorted list of `Event` values. The host receives these events and handles:
- Sample-accurate scheduling
- Synthesis / sample playback
- MIDI output
- Real-time audio I/O

The FFI can be modeled after Lua's C API: the host embeds the Tone interpreter, registers native functions (e.g., `render`, `export`), and evaluates Tone source code.

---

## 9. Future Directions

- **Operator overloading** for `seq` and `par` (e.g., `++` and `&`).
- **Module system** with `import`/`export` for organizing compositions and libraries.
- **REPL** for interactive exploration of tunings and patterns.
- **Live coding** support: re-evaluate definitions while playback continues.
- **Probability and randomness:** weighted choice, Markov chains for stochastic composition.
- **Spatial audio:** pan, distance, Ambisonics as `Control` modifiers.
- **Notation output:** export to LilyPond, MusicXML, or SVG staff notation.
- **Property-based testing** framework for verifying algebraic laws.
- **Type classes / traits** for generic numeric operations and custom interval types.

---

## Appendix A: Quick Reference Card

```
// types
let x: Ratio = 3/2          // exact rational
let y: Float = 1.5           // inexact float
let z = 3/2 + 1/2i           // complex with rational parts
let s = 7\12                  // EDO step literal
let f = 440.0hz               // frequency
let c = 701.955c              // cents

// functions
func name(arg: Type): ReturnType = single_expr
func name(arg: Type): ReturnType { multi; statement; body }

// lambdas
(x) => x * 2
(x, y) => { let z = x + y; z * 2 }

// generators
gen n in 1.. n * fundamental
gen n in range where cond { yield expr; yield expr2 }

// control flow
if cond { a } else { b }
for item in collection { body }
match value { Pattern => result }

// UFCS
notes.map(f).filter(g).take(16)  // == take(filter(map(notes, f), g), 16)

// music
note(pitch, duration)
a.seq(b)                          // sequential
a.par(b)                          // parallel
m.transpose(interval)
m.invert(axis)
m.retrograde
mix([voice(a), voice(b).offset(1/4)])
piece.realize.render(bpm: 120, ref: 262.0hz, sample_rate: 48000)
```
