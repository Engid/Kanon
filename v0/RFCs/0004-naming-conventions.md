# RFC-0004: Naming Conventions

**Status:** Draft  
**Created:** 2026-04-05  
**Related:** §2 Lexical Grammar, §5 Standard Library

---

## Summary

Establish naming conventions for Clef code: **PascalCase** for types and enum variants, **snake_case** for functions and variables. These conventions ensure visual distinction between types and values, and readable UFCS chains.

---

## Motivation

Consistent naming improves readability, makes the distinction between types and values immediate at a glance, and sets expectations for the stdlib that user code will follow.

---

## Proposed Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Types (product, enum, alias) | PascalCase | `Note`, `Music`, `Interval`, `RenderContext` |
| Enum variants | PascalCase | `Music.Note`, `Result.Ok`, `Control.Transpose` |
| Functions | snake_case | `build_triad`, `scale_tempo`, `take_beats` |
| Variables / bindings | snake_case | `my_scale`, `base_pitch`, `note_count` |
| Constants | snake_case or UPPER_SNAKE | TBD |
| Modules / files | snake_case | `scales.clef`, `tuning_tables.clef` |
| Type parameters | Single uppercase letter | `T`, `E`, `U` |

---

## Rationale

### PascalCase for Types

Universal convention across C#, TypeScript, Kotlin, Rust, Swift, Go. Non-negotiable. Creates a clear visual distinction between types and values.

### snake_case for Functions

Reads well in UFCS chains:

```
theme
    .transpose(6\31)
    .scale_tempo(3/4)
    .invert(0\31)
    .take_beats(16/1)
```

Compound names like `scale_tempo`, `take_beats`, `quantize_to` scan better in snake_case than `scaleTempo`, `takeBeats`, `quantizeTo`. The underscores act as visual word separators at a glance.

### The camelCase Alternative

Faster to type (no underscore key), familiar to JS/C#/Kotlin audience. The same chain in camelCase:

```
theme
    .transpose(6\31)
    .scaleTempo(3/4)
    .invert(0\31)
    .takeBeats(16/1)
```

Also reads fine. Not dramatically worse or better. This is genuinely a preference call.

---

## Disambiguation Rules

Whatever casing is chosen, these rules hold:

- **PascalCase in a value position is always a type constructor or enum variant.** `Note { ... }` constructs a Note, `Music.Seq(a, b)` constructs a variant.
- **lowercase/snake_case in a type position is never valid.** Types are always PascalCase.
- **This means you can always tell types from values at a glance.** `Ratio` is a type. `ratio()` is a function. `Note` is a type constructor. `note()` is a convenience function.

---

## Standard Library Naming

The stdlib should set the convention that user code follows. Current stdlib names in snake_case:

```
// sequence ops
map, filter, take, drop, reduce, fold, flat_map, take_while, sort_by

// music
note, rest, seq, par, transpose, invert, retrograde, scale_tempo

// tuning
edo, cents, ratio, freq, nearest, odd_limit

// math
abs, max, min, div, mod
```

Short, common functions are single words (no casing issue): `map`, `take`, `note`, `rest`. The convention only matters for multi-word names.

---

## Prior Art

| Language | Types | Functions/Variables | Notes |
|----------|-------|-------------------|-------|
| **Rust** | PascalCase | snake_case | Compiler warns on violations |
| **Go** | PascalCase | camelCase | Exported vs unexported via capitalization |
| **TypeScript** | PascalCase | camelCase | Convention, not enforced |
| **Python** | PascalCase | snake_case | PEP 8 convention |
| **Kotlin** | PascalCase | camelCase | Convention |
| **Swift** | PascalCase | camelCase | Convention |

Clef aligns with Rust and Python on snake_case for functions/variables.

---

## Open Questions

1. **Constants:** `UPPER_SNAKE` (like Rust/C) or just `snake_case` (like Python)? Constants are rare in a music DSL — maybe just snake_case and let it be.

2. **Acronyms in PascalCase:** Is it `EDO` or `Edo`? `MIDI` or `Midi`? `Hz` or `HZ`? Current spec uses `Edo`, `Hz`, `Cents` — treating them as regular words, not acronyms. This follows Go and Swift conventions.

3. **Enforced or conventional?** Should the compiler warn on casing violations (like Go does for exported names) or is it purely a style guide? Probably start as convention, add a linter later.

---

## Decisions Requested

- [ ] **Function casing** — Confirm snake_case over camelCase for functions/variables?
- [ ] **Constants** — `UPPER_SNAKE` or `snake_case`?
- [ ] **Acronyms** — `Edo`/`Hz` (word-style) or `EDO`/`HZ` (acronym-style)?
- [ ] **Enforcement** — Convention only, or compiler warnings?
