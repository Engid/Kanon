# Workshop: Naming Conventions

> **Created:** 2026-04-05  
> **Last Updated:** 2026-04-06  
> **Changelog:**
> - 2026-04-05 — Initial draft
> - 2026-04-06 — Converted to RFC-0004; added metadata headers

**Status:** Tentative. PascalCase for types is decided. Function/variable casing to be confirmed through real usage.

> **Note:** This workshop document has been converted to [RFC-0004](../RFCs/0004-naming-conventions.md). Future updates should be made to the RFC.

---

## Current Leanings

| Element | Convention | Example |
|---------|-----------|---------|
| Types (product, enum, alias) | PascalCase | `Note`, `Music`, `Interval`, `RenderContext` |
| Enum variants | PascalCase | `Music.Note`, `Result.Ok`, `Control.Transpose` |
| Functions | snake_case | `build_triad`, `scale_tempo`, `take_beats` |
| Variables / bindings | snake_case | `my_scale`, `base_pitch`, `note_count` |
| Constants | snake_case or UPPER_SNAKE | TBD |
| Modules / files | snake_case | `scales.clef`, `tuning_tables.clef` |
| Type parameters | Single uppercase letter | `T`, `E`, `U` |

## Rationale

**PascalCase for types** — universal convention across C#, TypeScript, Kotlin, Rust, Swift, Go. Non-negotiable. Creates a clear visual distinction between types and values.

**snake_case for functions** — reads well in UFCS chains:

```
theme
    .transpose(6\31)
    .scale_tempo(3/4)
    .invert(0\31)
    .take_beats(16/1)
```

Compound names like `scale_tempo`, `take_beats`, `quantize_to` scan better in snake_case than `scaleTempo`, `takeBeats`, `quantizeTo`. The underscores act as visual word separators at a glance.

**The camelCase alternative** — faster to type (no underscore key), familiar to JS/C#/Kotlin audience. The same chain in camelCase:

```
theme
    .transpose(6\31)
    .scaleTempo(3/4)
    .invert(0\31)
    .takeBeats(16/1)
```

Also reads fine. Not dramatically worse or better. This is genuinely a preference call.

## Disambiguation Rules

Whatever casing we choose, these rules should hold:

- **PascalCase in a value position is always a type constructor or enum variant.** `Note { ... }` constructs a Note, `Music.Seq(a, b)` constructs a variant.
- **lowercase/snake_case in a type position is never valid.** Types are always PascalCase.
- **This means you can always tell types from values at a glance.** `Ratio` is a type. `ratio()` is a function. `Note` is a type constructor. `note()` is a convenience function.

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

## Open Questions

- **Constants:** `UPPER_SNAKE` (like Rust/C) or just `snake_case` (like Python)? Constants are rare in a music DSL — maybe just snake_case and let it be.
- **Acronyms in PascalCase:** Is it `EDO` or `Edo`? `MIDI` or `Midi`? `Hz` or `HZ`? Current spec uses `Edo`, `Hz`, `Cents` — treating them as regular words, not acronyms. This follows Go and Swift conventions.
- **Enforced or conventional?** Should the compiler warn on casing violations (like Go does for exported names) or is it purely a style guide? Probably start as convention, add a linter later.
