# 9. Modules and Imports

> **Created:** 2026-04-05  
> **Last Updated:** 2026-04-06  
> **Changelog:**
> - 2026-04-05 — Initial draft
> - 2026-04-06 — Added metadata headers

## 9.1 Overview

Kanon's module system follows a simple principle: **the filesystem is the module tree.** A file is a module. A directory is a namespace. Imports are quoted paths. There is no separate module declaration — the file's location defines its identity.

## 9.2 File = Module

Every `.kan` file is a module. The module's name is derived from its filename (without the extension).

```
my-project/
├── main.kan                  // entry point
├── scales.kan                // module "scales"
├── tuning.kan                // module "tuning"
├── utils.kan                 // module "utils"
└── patterns/
    ├── arpeggios.kan         // module "patterns/arpeggios"
    ├── rhythms.kan           // module "patterns/rhythms"
    └── transforms.kan        // module "patterns/transforms"
```

There is no `module` or `package` declaration inside the file. The file's path relative to the project root IS its module identity.

## 9.3 Import Syntax

Imports use quoted string paths (Go-style). The quoted path makes parsing unambiguous and allows for URL-based external imports later.

### Basic Import

```
import "scales"
import "tuning"
import "patterns/arpeggios"
```

The imported module's name becomes a namespace. Access members with dot notation:

```
import "scales"

let my_scale = scales.major_31edo
let chord = scales.build_triad(my_scale, 0)
```

### Selective Import

Pull specific names directly into the current scope:

```
import "scales" { major_31edo, minor_31edo, build_triad }

// no prefix needed
let my_scale = major_31edo
let chord = build_triad(my_scale, 0)
```

### Wildcard Import

Import everything from a module into the current scope. Use sparingly — it pollutes the namespace.

```
import "scales" { * }

let s = major_31edo   // directly available
```

### Aliasing

Rename a module on import:

```
import "patterns/arpeggios" as arp

let melody = arp.up_down(scale, 1/8)
```

Aliasing also works with selective imports:

```
import "scales" { major_31edo as major, minor_31edo as minor }

let s = major
```

### Multiple Imports

```
import "scales"
import "tuning"
import "patterns/arpeggios" as arp
import "patterns/transforms" { invert, retrograde }
```

## 9.4 Visibility

**Everything is public by default.** All top-level declarations in a file (functions, types, enums, let bindings) are importable by other modules.

Use the `private` keyword to hide declarations:

```
// scales.kan

// public — importable by other modules
fun major_31edo(): list[Interval] {
    [0\31, 5\31, 10\31, 13\31, 18\31, 23\31, 28\31]
}

// private — only visible within this file
private fun reduce_to_octave(interval: Ratio): Ratio {
    // helper function, not part of the public API
    // ...
}

// public type
type ScaleDefinition {
    name: string,
    steps: list[Interval],
    period: Ratio,
}

// private type
private type InternalCache {
    computed: Map[string, list[Interval]],
}
```

### Rationale

Most Kanon users are composers sharing code with each other, not enterprise teams building layered architectures. Making everything public by default reduces friction — you define a function, it's immediately usable by anyone who imports your file. The `private` escape hatch exists for when you genuinely need to hide implementation details.

## 9.5 UFCS and Import Scope

**A function is only usable as a UFCS method if it is in the current scope** — either defined in the current file, or explicitly imported.

This prevents the "phantom method" problem where a transitive dependency adds a function that suddenly appears as a method on your types.

```
// transforms.kan
fun quantize_to(seq: Seq[Note], scale: list[Interval]): Seq[Note] {
    seq.map(n => Note { ...n, pitch: nearest(scale, n.pitch) })
}
```

```
// main.kan

import "scales"

// quantize_to is NOT available as a method — it's not imported
melody.quantize_to(scales.major)   // ERROR: no method quantize_to on Seq[Note]

// after importing:
import "transforms" { quantize_to }

melody.quantize_to(scales.major)   // OK — quantize_to is now in scope
```

This is the same model as C# extension methods: they only activate when their containing namespace is imported with `using`.

## 9.6 No Circular Imports

If module A imports module B, then module B cannot import module A (directly or transitively). The dependency graph must be a DAG (directed acyclic graph).

```
// scales.kan
import "tuning"         // OK

// tuning.kan
import "scales"         // ERROR: circular import (scales → tuning → scales)
```

If two modules need to share definitions, extract the shared parts into a third module that both can import.

This constraint keeps compilation order simple and prevents tightly coupled module tangles.

## 9.7 External Dependencies

External code is imported by git URL:

```
import "github.com/someone/xenharmonic-scales"
import "github.com/someone/xenharmonic-scales" as xen

let bp = xen.bohlen_pierce
```

### Resolution

When the compiler encounters a URL-style import:

1. Check the local cache (`~/.kanon/cache/` or a project-local `.kanon-cache/`)
2. If not cached, clone/fetch the repository
3. Pin the resolved commit hash in a lockfile (`kanon.lock`)

### Lockfile

The lockfile ensures reproducible builds. It records the exact commit used for each external dependency.

```toml
# kanon.lock — auto-generated, commit to version control

[dependencies]

[dependencies."github.com/someone/xenharmonic-scales"]
url = "https://github.com/someone/xenharmonic-scales.git"
commit = "a1b2c3d4e5f6"
fetched = "2026-03-15T10:30:00Z"

[dependencies."github.com/other/cool-patterns"]
url = "https://github.com/other/cool-patterns.git"
commit = "f6e5d4c3b2a1"
fetched = "2026-03-20T14:00:00Z"
```

### Update Command

```bash
kanon update                              # update all deps to latest
kanon update "github.com/someone/repo"    # update one dep
kanon update --lock                       # just regenerate lockfile from cache
```

### No Package Registry (v0.1)

There is no centralized package registry in v0.1. Dependencies are fetched directly from git. A registry (like crates.io or npm) can be added later when the community is large enough to need one.

This avoids the supply-chain attack surface of centralized registries while still making it easy to share code. The tradeoff is discoverability — finding packages requires knowing the URL. A community-maintained list or search tool can fill this gap without building infrastructure.

## 9.8 Standard Library

The standard library is divided into a **core** (always available, no import needed) and **extended modules** (require explicit import).

### Core (Always Available)

These functions and types are in scope in every Kanon file without any import:

- **Basic types:** `Int`, `Ratio`, `Float`, `string`, `bool`, `nil`
- **Music types:** `Music`, `Control`, `Interval`, `Edo`, `Cents`, `Hz`
- **Collection types:** `list`, `Seq`, `Map`
- **Music primitives:** `note`, `rest`, `seq`, `par`
- **Sequence essentials:** `map`, `filter`, `take`, `drop`, `reduce`, `fold`, `collect`, `cycle`, `repeat`, `zip`, `flatten`
- **Basic math:** `abs`, `max`, `min`
- **Conversion:** `cents`, `ratio`, `freq`, `float`
- **I/O:** `print`

### Extended Modules (Import Required)

```
import "std/math"       // sin, cos, sqrt, pow, log, log2, floor, ceil, round
import "std/tuning"     // edo, limit, odd_limit, nearest, mode
import "std/music"      // transpose, invert, retrograde, scale_tempo,
                        // realize, duration, pitches, mix, voice
import "std/midi"       // to_midi, from_midi
import "std/random"     // (future) rand, weighted_choice, markov
```

### Rationale

The core set covers what you need in ~90% of Kanon files. Specialized functionality (advanced math, tuning theory, MIDI, randomness) is one import away. This keeps the global namespace manageable while avoiding boilerplate at the top of every file.

## 9.9 Project Structure

A Kanon project has a root directory containing a `kanon.toml` manifest (optional for single-file scripts, required for multi-file projects):

```toml
# kanon.toml

[project]
name = "my-composition"
version = "0.1.0"
entry = "main.kan"

[dependencies]
"github.com/someone/xenharmonic-scales" = { branch = "main" }
"github.com/other/cool-patterns" = { tag = "v1.2.0" }
```

For single-file scripts, no manifest is needed:

```bash
kanon run my_sketch.kan        # just run it
```

For multi-file projects:

```bash
kanon run                      # reads kanon.toml, runs entry point
kanon check                    # type-check without running
kanon update                   # update external dependencies
```

## 9.10 Grammar Additions

```ebnf
import_decl    = "import" STRING [ "as" IDENT ]
               | "import" STRING "{" import_list "}" ;

import_list    = "*"
               | import_item { "," import_item } [ "," ] ;

import_item    = IDENT [ "as" IDENT ] ;

visibility     = [ "private" ] ;

fun_decl       = visibility "fun" IDENT ... ;
type_decl      = visibility "type" IDENT ... ;
enum_decl      = visibility "enum" IDENT ... ;
let_decl       = visibility ( "let" | "mut" ) IDENT ... ;
```
