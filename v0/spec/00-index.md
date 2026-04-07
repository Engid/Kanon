# Kanon Language Specification — Index

**Version 0.2 — Draft**

> **Created:** 2026-04-05  
> **Last Updated:** 2026-04-06  
> **Changelog:**
> - 2026-04-05 — Initial draft
> - 2026-04-06 — Added metadata headers

## Document Map

| File | Contents |
|------|----------|
| `00-index.md` | This file — glossary, concept index, quick reference |
| `01-overview.md` | Philosophy, design principles, architecture, name |
| `02-lexical-grammar.md` | Tokens, keywords, numeric literals, operators, comments |
| `03-type-system.md` | Numeric tower, product types, sum types, aliases, Music type, algebraic laws |
| `04-expressions.md` | Variables, functions, lambdas, control flow, match, generators, UFCS |
| `05-stdlib.md` | Sequence ops, math, music construction, mixing/voices, rendering |
| `06-formal-grammar.md` | EBNF grammar, Pratt parser precedence table, parsing notes |
| `07-examples.md` | Complete example programs |
| `08-implementation.md` | Runtime, GC, host FFI, future directions |

## Concept Index

| Concept | Primary Location | Also Mentioned In |
|---------|-----------------|-------------------|
| Algebraic laws | 03 §3.5 | 07 §7.5, 07 §7.7 |
| Aliases (type) | 03 §3.4 | 06 |
| Architecture (host/guest) | 01 §1.2 | 08 §8.3 |
| Braces (mandatory for control flow) | 04 §4.4 | 06 |
| Cents (unit type) | 02 §2.4, 03 §3.2 | 05 §5.5 |
| Complex numbers | 02 §2.4 | 03 §3.1, 07 §7.8 |
| Copy-with (`.with()`) | 03 §3.3 | 04 §4.1 |
| EDO literals (`n\d`) | 02 §2.4 | 03 §3.2, 07 §7.3–7.6 |
| Enum (sum types) | 03 §3.3 | 06 |
| Event type | 05 §5.7 | 08 §8.3 |
| Exactness (Racket model) | 03 §3.1 | 02 §2.4 |
| Fat-arrow lambdas | 04 §4.3 | 06 |
| Floor division (`div`/`mod`) | 05 §5.3 | 02 §2.6 |
| For loops (imperative) | 04 §4.4 | 06 |
| Frequency (Hz) | 02 §2.4, 03 §3.2 | 05 §5.5 |
| `fun` keyword | 04 §4.2 | 06 |
| Generators (`gen`) | 04 §4.6 | 06, 07 §7.3–7.4 |
| Generics (`[]` brackets) | 03 §3.3 | 06 |
| Host FFI | 08 §8.3 | 01 §1.2 |
| Indexing (parenthesis-based) | 04 §4.7, 08 §8.2 | 06 |
| Interval (abstract type) | 03 §3.2 | 05 §5.5 |
| Match expressions | 04 §4.5 | 06, 07 §7.7 |
| Mix (voice combining) | 05 §5.6 | 07 §7.4 |
| Music (algebraic type) | 03 §3.4 | 05 §5.4, 07 §7.5, 07 §7.7 |
| Numeric tower | 03 §3.1 | 02 §2.4 |
| Pattern matching | 04 §4.5 | 03 §3.3, 07 §7.7 |
| Pipe operator (`\|>`) | 04 §4.7 | 02 §2.6, 06 |
| Pratt parser | 06 §6.1 | 08 §8.1 |
| Rational literals (`/`) | 02 §2.4 | 03 §3.1, 06 §6.1 |
| Realize (Music → Events) | 05 §5.7 | 08 §8.3 |
| Seq (lazy sequences) | 04 §4.6 | 05 §5.1 |
| Structural equality | 03 §3.3 | |
| Type (product types) | 03 §3.3 | 06 |
| UFCS | 04 §4.7 | 01, 05, 06 |
| Unit types (Hz, Cents) | 02 §2.4 | 03 §3.2 |
| Value semantics | 03 §3.3 | 01 |
| Voice (mix component) | 05 §5.6 | 07 §7.4 |
| `yield` keyword | 04 §4.6 | 06 |

## Quick Reference Card

```
// --- Types ---
type Note {                              // product type (immutable)
    pitch: Interval,
    duration: Ratio,
    velocity: number,
}

enum Music {                             // sum type
    Note(pitch: Interval, duration: Ratio, velocity: number),
    Rest(duration: Ratio),
    Seq(a: Music, b: Music),
    Par(a: Music, b: Music),
}

type Scale = list[Interval]              // type alias (uses =)

// --- Literals ---
let r = 3/2                              // exact rational
let f = 1.5                              // inexact float
let z = 3/2 + 1/2i                       // complex
let s = 7\12                             // EDO step
let freq = 440.0hz                       // frequency
let c = 701.955c                         // cents

// --- Functions ---
fun name(arg: Type): ReturnType = expr            // single-expression
fun name(arg: Type): ReturnType { body }          // multi-statement

// --- Lambdas ---
(x) => x * 2                                      // single-expression
(x, y) => { let z = x + y; z * 2 }               // multi-statement

// --- Generators ---
gen n in 1.. n * fundamental                       // implicit yield
gen n in range where cond { yield expr }           // explicit yield

// --- Control Flow ---
if cond { a } else { b }                           // always braced
for item in collection { body }                    // imperative loop
match value {                                      // pattern matching
    Pattern(x) => result,                          // comma-separated arms
    _ => default,
}

// --- UFCS ---
notes.map(f).filter(g).take(16)                    // free functions as methods

// --- Indexing ---
scale(0)                                           // parenthesis indexing (not [])
notes(3)                                           // [] is only for type parameters

// --- Music ---
note(pitch, duration)                              // construct a Music.Note
a.seq(b)                                           // sequential composition
a.par(b)                                           // parallel composition
m.transpose(interval)                              // transformation
mix([voice(a), voice(b).offset(1/4)])              // combine voices
piece.realize.render(bpm: 120, ref: 262.0hz, sample_rate: 48000)
```
