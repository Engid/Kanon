# 1. Overview

> **Created:** 2026-04-05  
> **Last Updated:** 2026-04-06  
> **Changelog:**
> - 2026-04-05 — Initial draft
> - 2026-04-06 — Added metadata headers
> - 2026-04-06 — Renamed the language to Kanon, updated `.kan`, and added the naming rationale

## 1.1 What Is Kanon?

Kanon is a domain-specific language for composing microtonal music. It provides first-class rational numbers, lazy generators, algebraic music types, and C-family syntax in a garbage-collected, interpreted runtime.

The name "Kanon" refers to the Pythagorean monochord, the single-string instrument used to demonstrate that consonant musical intervals arise from exact numerical ratios. It fits a language built around exact rationals, interval structure, and composition as mathematics made audible. The name also resonates with the musical idea of a canon and with the modern use of Kanon as a musical given name.

## 1.2 Design Principles

**Numerical precision by default.** Rational numbers are exact. Floating-point is opt-in. Just intonation intervals are ratios, not approximations. EDO steps, cents, and frequencies have dedicated literal syntax.

**Composability.** Everything composes: functions via UFCS, sequences via generators, musical structures via algebraic operators. User-defined operations are indistinguishable from built-ins. This follows Guy Steele's growability principle: a language should start small, and user-defined abstractions should look and behave like primitives.

**Familiarity.** C-family syntax with TypeScript-style type annotations. Fat-arrow lambdas. Curly braces. `fun` for function declarations. A JS/C#/Kotlin developer should be able to read Kanon code on day one.

**Music-native types.** EDO steps, cents, frequencies, and intervals are first-class with literal syntax and unit safety. The core `Music` algebraic type supports structural composition, transformation, and analysis.

**Immutability by default.** All values have value semantics. Product types are immutable. Copy-with-modification via `.with()`. Mutable bindings exist (`mut`) but mutable fields on types do not (yet).

## 1.3 Architecture

Kanon is designed to be embedded in a host audio engine, analogous to how Lua, sclang (SuperCollider), or Python are used in audio systems. The host handles real-time sample-level rendering; Kanon handles composition, pattern generation, and structural manipulation.

```
┌─────────────────────────────┐
│  Kanon Source Code           │
│  (.kan files)                │
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

The boundary between Kanon and the host is the `Event` type — a flat, time-sorted list of `(time, pitch, duration, velocity, instrument)` tuples. The `realize` function walks a `Music` tree and produces this event list. The host consumes it for synthesis, MIDI output, or notation rendering.

## 1.4 Influences

| Influence | What Kanon borrows |
|-----------|-------------------|
| Racket/Scheme | Exact-by-default numeric tower, rational literal syntax |
| JavaScript/TypeScript | Fat-arrow lambdas, type annotation style, `[]` generics |
| Kotlin | `fun` keyword, expression-oriented `if`/`match`, single-expression function bodies with `=` |
| C#  | Fat-arrow lambdas, value-oriented records, LINQ-style chaining |
| D/Nim | UFCS (Uniform Function Call Syntax) |
| Haskell (Euterpea/Haskore) | Algebraic `Music` type, sequential/parallel composition, algebraic laws |
| SuperCollider | Music DSL domain, pattern generators, audio architecture |
| Xenharmonic community | EDO backslash notation (`7\12`, `9\31`) |
| Guy Steele ("Growing a Language") | Growability — user-defined abstractions indistinguishable from built-ins |
| Bob Nystrom (Crafting Interpreters) | C-family syntax for approachability, Pratt parsing for expression grammar |
| Paul Hudak (Haskore/Euterpea) | Algebraic music semantics, equational laws for composition operators |
