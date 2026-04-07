# 10. Sketch Notes and Open Questions

> **Created:** 2026-04-05  
> **Last Updated:** 2026-04-06  
> **Changelog:**
> - 2026-04-05 — Initial draft
> - 2026-04-06 — Added metadata headers

This document collects design discussions, unresolved questions, and ideas for future consideration. Items here are NOT part of the specification — they're conversation notes to revisit later.

---

## Naming

**Current name:** Kanon.

Kanon is the Greek name of the Pythagorean monochord, the single-string instrument used to demonstrate that consonant intervals arise from exact whole-number ratios. That makes it a direct fit for a language built around exact rationals, tunings, and compositional structure.

**Why it works:**
- It ties the project directly to the historical discovery that music and mathematics are linked by ratio.
- It matches the language's emphasis on exact numeric relationships, especially rational intervals.
- It also echoes the musical idea of a canon, which fits layering and offset composition.
- Kanon is also used as a Japanese given name with musical associations, which gives the name an additional musical resonance.

**Decision:** The language name is Kanon. The file extension is `.kan`. Tooling and project artifacts use the `kanon` prefix.

---

## Indexing Syntax: `[]` vs `()`

**Current decision:** Parenthesis-based indexing (Scala-style). `scale(0)` not `scale[0]`. Square brackets are exclusively for type parameters and list literals.

**Open concern:** Using `[]` for list literals but not indexing feels asymmetric. A list constructed with `[1, 2, 3]` is accessed with `(0)`. This is internally consistent (a list is a function from index to value) but might surprise JS/Python/C# users who expect `[0]` for indexing.

**Alternatives to revisit:**
- Use `()` for everything including list construction: `list(1, 2, 3)` — but loses the visual distinction of literal syntax
- Use `[]` for indexing too, disambiguate from generics contextually (the Scala approach, which has worked in practice for 20+ years)
- Use a different bracket for generics entirely: angle brackets `<>` (Java/C#/TypeScript) or backtick-delimited (`Seq\`Note\``)

**Parking this.** The Scala precedent shows `[]` for both generics and indexing works if the parser knows the context. We could switch back if `()` indexing proves confusing in practice.

---

## Lambda Syntax: `=` vs `=>` for Function Bodies

**Current decision:** `fun f(): T = expr` for single-expression functions, `(x) => expr` for lambdas.

**Concern raised:** `=` and `=>` look similar. Could be confusing.

**Counter-argument:** They appear in different syntactic contexts (`=` after a typed function signature, `=>` after a parameter list). The parser has no ambiguity. Kotlin uses exactly this split successfully.

**Alternative considered:** Always use braces for named functions (no `=` shorthand). `fun f(): T { expr }` where the last expression is the return value. Slightly more verbose but avoids the `=` / `=>` question entirely.

---

## Operator Overloading for `seq` and `par`

**Deferred to future version.** Currently using UFCS: `a.seq(b)` and `a.par(b)`.

**Candidate operators:**
- `++` for seq, `&` for par
- `++` for seq, `<|>` for par  
- Custom operators via a trait/protocol system

**Concern:** `&` collides with potential bitwise AND. `||` collides with logical OR. `<|>` is Haskell-flavored and unfamiliar to C-family users.

**Recommendation when we get here:** Implement a lightweight operator overloading system via traits. Define a `Sequenceable` trait with an associated `++` operator and a `Parallelizable` trait with `&` or similar. This keeps it growable (Steele principle) without hardcoding operators for just `Music`.

---

## Generator Keyword: `gen` vs `for`/`yield`

**Current decision:** `gen` for generators, `for` for imperative loops. Distinct keywords, no ambiguity.

**Alternative considered:** Using `for` for both, with `yield` presence determining if it's a generator. Rejected because the compiler would need to scan the entire body to determine semantics, and it's not obvious at the call site.

**`yield` keyword:** Sticking with `yield` (well-established in Python, C#, JS). Considered `emit` (more evocative for music/audio) but familiarity wins.

---

## Floor Division

**Current decision:** `div()` and `mod()` as stdlib functions, chainable via UFCS: `7.div(2)`.

**Why not `//`:** `//` is used for comments (C-family familiarity).

**Why not a keyword:** `div` could be a keyword (like Pascal/Haskell), but as a function it's simpler — no grammar changes, works with UFCS, consistent with the "stdlib function" philosophy.

---

## Value Semantics and Copy-on-Write

**Current decision:** All types are immutable, value semantics. Implementation uses Arc (ref-counted) with COW. `.with()` for copy-with-modification.

**Open question:** Do we ever need mutable fields? For a composition DSL, probably not in v0.1. If we do, the path is adding `mut` as a field modifier: `type Synth { mut frequency: Hz }`. This is a backwards-compatible addition.

**Open question:** Should `.with()` be a keyword/syntax or a method? Current design uses it as a method. Alternative: spread syntax like `Note { ...n, pitch: 9\12 }` (JS-style). Could support both.

---

## Reference Counting vs. GC

**Current decision:** Reference counting with Arc (native) / Rc (WASM). No tracing GC.

**Cycle risk:** Music trees are acyclic by construction. Closures can theoretically cycle but rarely do in practice. If cycles become an issue, add an optional cycle collector invokable by the host.

**Performance note:** Arc has ~30% overhead vs raw pointers for operations that frequently clone/drop references. For a tree-walk interpreter this is negligible — the interpreter loop itself is the bottleneck, not refcounting. If/when we move to a bytecode VM, we can revisit.

---

## Module System: External Dependencies

**Current decision:** Import by git URL, pin with a lockfile, no registry.

**Open question:** Should we support non-git sources? (zip archives, local paths for development?) Local paths are probably needed for the development workflow:

```
import "../../my-local-lib"    // relative path
```

**Open question:** Version constraints. The lockfile pins commits, but should `kanon.toml` support version ranges like `>= 1.0, < 2.0`? Probably yes eventually, but not in v0.1. For now, pin to a branch or tag.

**Ginger Bill's anti-package-manager stance (Odin):** Worth considering. His argument is that package managers create more problems than they solve (supply chain attacks, dependency hell, leftpad incidents). The git-URL approach sidesteps some of these by making dependencies explicit and auditable. A curated "awesome-kanon" list could serve as discovery without needing infrastructure.

---

## Type System: Traits / Protocols

**Not in v0.1.** But we'll want them eventually for:
- Operator overloading (defining `++` for custom types)
- Generic constraints (`fun sort[T: Comparable](...)`)
- Shared behavior across types (e.g., `Realizable` trait for things that can be converted to events)

**Candidate syntax (future):**
```
trait Pitched {
    fun pitch(self): Interval
    fun transpose(self, interval: Interval): Self
}
```

---

## Hosting: Browser Playground

**Target:** A web-based playground where users can write Kanon code, hear it rendered via Web Audio, and see visualizations of the Music tree and piano roll.

**Tech stack sketch:**
- Monaco editor (VS Code's editor component) for code editing
- Kanon runtime compiled to WASM
- Web Audio API for sound output (OscillatorNode for simple synthesis, AudioWorklet for custom)
- Canvas or WebGL for visualization (piano roll, Music tree, waveform)
- Maybe Tone.js as a higher-level audio framework

**Open question:** How to handle the "render" step. Options:
1. Kanon `realize` produces events → JS schedules them via Web Audio (simplest)
2. Kanon `realize` + a built-in Kanon synthesizer compiled to WASM (more self-contained, but complex)
3. Host provides a synthesis function back to Kanon via registered host functions (most flexible)

Probably start with option 1.

---

## Hosting: DAW Product

**Vision:** A special-purpose DAW with Kanon scripting built in. Proprietary, paid product. Kanon itself is open source; the DAW is commercial.

**Key features to support:**
- Live coding: re-evaluate on keystroke, hear changes immediately
- Visual feedback: Music tree rendered as a node graph, piano roll synced to playback
- Preset library: curated Kanon scripts for common patterns/tunings
- Plugin API: Kanon scripts as audio/MIDI effect plugins within the DAW
- Export: MIDI, audio, notation (LilyPond/MusicXML)

**Business model consideration:** Kanon is MIT/Apache licensed. The DAW is proprietary. Third-party hosts can also be commercial. This is the Redis/MongoDB/Elastic model — open core language, commercial tools built on top.

---

## Misc Notes

- **Erik Meijer** (not Eric Myers) — the Haskell/LINQ/Rx guy at Microsoft who's enthusiastic about Kotlin. His work on LINQ (monadic comprehension syntax in C#) is directly relevant to Kanon's generator/pipeline design.

- **Pratt parsing** — already planned, well-suited for the expression grammar. Bob Nystrom's blog post and the Crafting Interpreters chapter are the canonical tutorials.

- **Hudak's algebraic music model** — doesn't require Haskell features. Just needs: enum types (sum types), pattern matching, and recursive data structures. All of which Kanon has. The equational laws are properties of the implementation, not the type system.

- **SuperCollider** — NOT true UFCS. SC is Smalltalk-derived message-passing. You must add methods to classes to chain them with dot syntax. True UFCS (D, Nim, Koka, Lean) lets any free function be called as a method.
