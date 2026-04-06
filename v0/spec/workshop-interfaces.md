# Workshop: Interfaces and Structural Matching

> **Created:** 2026-04-05  
> **Last Updated:** 2026-04-06  
> **Changelog:**
> - 2026-04-05 — Initial draft
> - 2026-04-06 — Converted to RFC-0003; added metadata headers

**Status:** Design sketch. Not needed for v0.1 — `type`, `enum`, and free functions cover the initial use cases. Interfaces become useful for platform/host interaction and generic library code.

> **Note:** This workshop document has been converted to [RFC-0003](../RFCs/0003-interfaces.md). Future updates should be made to the RFC.

---

## The Keyword

```
interface Pitched {
    fun pitch(self): Interval,
    fun transpose(self, interval: Interval): Self,
}
```

Three type declaration keywords, three jobs:
- `type` — concrete product types (fields with data)
- `enum` — tagged unions (variants)
- `interface` — named sets of required function signatures

## Structural Matching

A type satisfies an interface if the required functions exist in scope with matching signatures. No `implements` keyword. The compiler checks the shape.

```
interface Pitched {
    fun pitch(self): Interval,
    fun transpose(self, interval: Interval): Self,
}

type Note {
    pitch: Interval,
    duration: Ratio,
}

// these functions give Note the right shape
fun pitch(n: Note): Interval = n.pitch
fun transpose(n: Note, interval: Interval): Note =
    n.with(pitch: n.pitch + interval)

// Note satisfies Pitched — checked by the compiler, no declaration needed
fun show_pitch(p: Pitched): String = "pitch: ${p.pitch()}"

show_pitch(Note { pitch: 7\12, duration: 1/4 })  // works
```

## How It Interacts with UFCS

This is where interfaces and UFCS reinforce each other naturally. A free function `fun pitch(n: Note): Interval` is:
1. Callable as `n.pitch()` via UFCS
2. Counts toward satisfying `Pitched` which requires `fun pitch(self): Interval`

Same function, two roles, zero extra syntax.

## How It Interacts with Imports

A type only satisfies an interface using functions that are **in scope**. If `transpose` is defined in another module, you must import it for the match to work:

```
// transforms.clef
fun transpose(n: Note, interval: Interval): Note = ...

// main.clef
import "transforms" { transpose }

// NOW Note satisfies Pitched (because transpose is in scope)
// Without the import, it wouldn't
```

This prevents phantom interface satisfaction from transitive dependencies — same rule as UFCS scoping (§9.5).

## Use Cases

### Platform / Host Interfaces

The primary motivation. A host defines an interface that plugins or scripts must satisfy:

```
interface AudioRenderer {
    fun render(self, events: List[Event], sample_rate: Int): Audio,
    fun supported_formats(self): List[String],
}

interface MidiOutput {
    fun send_note(self, channel: Int, pitch: Int, velocity: Int): Result[Nil, String],
    fun send_cc(self, channel: Int, cc: Int, value: Int): Result[Nil, String],
}
```

### Generic Library Functions

Write functions that work on "anything with a pitch" rather than a specific type:

```
fun transpose_all(items: List[Pitched], interval: Interval): List[Pitched] =
    items.map(p => p.transpose(interval))
```

### Custom Extension Points

A scale library could define an interface for custom tuning systems:

```
interface TuningSystem {
    fun step_count(self): Int,
    fun step_to_cents(self, step: Int): Cents,
    fun nearest_step(self, cents: Cents): Int,
}
```

Anyone can create a type that satisfies `TuningSystem` without modifying the library.

## Precedent

| Language | Keyword | Matching | Explicit Declaration |
|----------|---------|----------|---------------------|
| Go | `interface` | Structural | No |
| TypeScript | `interface` / `type` | Structural | No |
| Rust | `trait` | Nominal | Yes (`impl Trait for Type`) |
| C# | `interface` | Nominal | Yes (`class Foo : IBar`) |
| Swift | `protocol` | Nominal | Yes (`struct Foo: Bar`) |
| **Clef** | `interface` | **Structural** | **No** |

Clef follows the Go/TypeScript camp: structural matching, no explicit declaration.

## What's NOT Here

- **Default method implementations.** An interface is purely a signature contract. If you want shared behavior, write a free function that takes the interface type. This can be revisited later.
- **Interface inheritance / composition.** Could add later: `interface Transposable : Pitched { ... }`. Not needed for v0.1.
- **Associated types.** e.g., `interface Collection { type Element; ... }`. Advanced feature, park it.
- **Object literals.** Anonymous values that satisfy interfaces without a named type. Interesting but adds type system complexity. Defer.

## Open Questions

- **Runtime or compile-time checking?** In the v0.1 tree-walk interpreter, structural matching would be checked at call time (runtime). A future compiler would check at compile time. Both work — runtime is simpler to implement.
- **Error messages.** When a type doesn't satisfy an interface, the error should say exactly which function is missing or has the wrong signature. Good errors matter a lot here.
- **Self type.** `Self` in interface signatures refers to the implementing type. `fun transpose(self, interval: Interval): Self` means "returns the same type as the receiver." This requires some type system support to track.
