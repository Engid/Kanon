# 8. Implementation

## 8.1 Parsing Strategy

Use a hand-rolled recursive descent parser with Pratt parsing for expressions (see §6 for the full grammar and precedence table).

The scanner produces tokens; the parser consumes them to build an AST. A single-pass parser is sufficient for Clef's grammar — there are no forward references that require multiple passes (type declarations can be used before they appear in the source via a pre-pass or lazy resolution).

**Key parsing challenges:**

- **`/` as rational literal vs. division:** Check if both operands are integer literals after parsing a multiplication-precedence expression. See §6.3 for details.
- **`\` as EDO literal:** Highest precedence, both operands must be integer literals.
- **`i` as imaginary suffix:** Postfix on numeric expressions only.
- **`[]` is always generics.** Square brackets only appear in type positions. Indexing uses parentheses: `list(0)`, `scale(2)`. This eliminates all `[]` disambiguation.
- **`()` after a value:** Either a function call or an index. The type system disambiguates — calling a `list[T]` with an `Int` argument is indexing. From the parser's perspective, both are `CallExpr`; the interpreter resolves which operation to perform.
- **`gen` vs `for`:** `gen` always starts a generator expression; `for` always starts an imperative loop.
- **`type` with `=` vs `{}`:** `=` means alias; `{` means product type definition.
- **Type constructor vs block (`Ident {`):** If the content starts with `Ident ":"`, it's a type constructor.

**Recommended resource:** Bob Nystrom's *Crafting Interpreters* (craftinginterpreters.com) covers recursive descent and Pratt parsing in detail.

---

## 8.2 Runtime Architecture

### Overview

Clef v0.1 is a **tree-walk interpreter written in Rust**. This is the fastest path to a working implementation. A bytecode VM can be added later as an optimization (following the Crafting Interpreters progression: tree-walk first, bytecode second).

The runtime has these components:

```
┌─────────────────────────────────────────────────┐
│                  ClefState                       │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Scanner  │→│  Parser  │→│  Interpreter   │  │
│  │ (tokens) │  │  (AST)   │  │  (tree-walk)  │  │
│  └──────────┘  └──────────┘  └───────┬───────┘  │
│                                      │          │
│  ┌──────────────────┐  ┌─────────────▼────────┐ │
│  │  Environment     │  │  Value Store         │ │
│  │  (scope chain)   │  │  (Arc<Value> heap)   │ │
│  └──────────────────┘  └──────────────────────┘ │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  Host Function Registry                   │   │
│  │  (native functions registered by host)    │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### Value Representation

All Clef values are represented as a Rust enum wrapped in `Arc` (atomic reference counting) for shared ownership:

```rust
use std::sync::Arc;
use num::{BigInt, BigRational};

#[derive(Clone, Debug)]
pub enum Value {
    Nil,
    Bool(bool),
    Int(i64),
    BigInt(Arc<BigInt>),
    Ratio(Arc<BigRational>),
    Float(f64),
    Complex(Box<Value>, Box<Value>),
    Edo(i32, i32),                 // step, divisions
    Cents(f64),
    Hz(f64),
    Str(Arc<str>),
    List(Arc<Vec<Value>>),
    Music(Arc<MusicNode>),
    Closure(Arc<Closure>),
    Generator(Arc<GeneratorState>),
    HostFn(Arc<HostFunction>),
    TypeInstance(Arc<TypeInstance>),
    EnumVariant(Arc<EnumInstance>),
}

#[derive(Clone, Debug)]
pub enum MusicNode {
    Note { pitch: Value, duration: Value, velocity: Value },
    Rest { duration: Value },
    Seq(Arc<MusicNode>, Arc<MusicNode>),
    Par(Arc<MusicNode>, Arc<MusicNode>),
    Modify(ControlNode, Arc<MusicNode>),
}
```

**Why `Arc` everywhere:** Reference counting gives deterministic deallocation — objects are freed the instant their last reference drops. No stop-the-world GC pauses, which is critical for audio host environments where latency spikes are unacceptable.

**Structural sharing:** `Music` trees naturally share structure. `a.seq(b)` creates one new `Seq` node pointing to existing `Arc<MusicNode>` values for `a` and `b`. No copying of subtrees. Transformations like `theme.transpose(3\12).seq(theme.retrograde)` are cheap — each wraps or rearranges `Arc` pointers, not data.

### Reference Counting Strategy

**Advantages over tracing GC:**
- Deterministic: no pause times, no latency spikes
- Simple: Rust's `Arc` handles it automatically
- FFI-friendly: hosts hold `Arc` references (exposed as opaque pointers via C API)
- WASM-friendly: no GC runtime needed in the WASM module

**The cycle problem:** Reference cycles cause memory leaks with pure refcounting. For Clef, this is manageable:

- `Music` trees are **acyclic by construction** — recursive algebraic types where nodes point to children, never parents
- `List` and `TypeInstance` values are acyclic
- `Closure` values *can* theoretically cycle, but this is rare in a composition DSL
- **Mitigation:** Add an optional cycle collector the host can invoke explicitly during non-real-time moments (e.g., between audio buffer fills, or on user request)

### Copy-on-Write

When `.with()` is called on a product type instance:

```rust
impl TypeInstance {
    pub fn with_field(&self, field: &str, new_value: Value) -> TypeInstance {
        // Clone the fields vec. If this TypeInstance's Arc refcount is 1,
        // Rust will optimize this to a move rather than a copy.
        let mut new_fields = self.fields.clone();
        new_fields.set(field, new_value);
        TypeInstance { type_name: self.type_name.clone(), fields: new_fields }
    }
}
```

In pipelines like `note.with(pitch: x).with(duration: y)`, each intermediate value is consumed immediately (refcount = 1), so the clone is optimized away by Rust.

### Rational Arithmetic

Exact rationals use Rust's `num` crate (`BigRational` backed by `BigInt`). Auto-simplification (GCD reduction) happens on every operation.

```rust
use num::BigRational;

fn ratio_from_literal(numer: i64, denom: i64) -> BigRational {
    BigRational::new(BigInt::from(numer), BigInt::from(denom))
    // auto-reduces: 6/4 becomes 3/2
}
```

**Performance note:** Extended just intonation calculations can produce rationals with very large numerators/denominators. A `limit_denominator(max)` function should be in the standard library for approximating large rationals.

### Lazy Sequences (Generators)

Generators are **closure-based iterators**. Each `gen` block compiles to a closure that captures iteration state.

```rust
pub struct GeneratorState {
    next_fn: Box<dyn FnMut() -> Option<Value>>,
}

impl GeneratorState {
    pub fn next(&mut self) -> Option<Value> {
        (self.next_fn)()
    }
}
```

Multi-yield generators (with explicit `yield`) are desugared into a state machine at the AST level. Each `yield` point becomes a state, and the generator closure switches between states on each `.next()` call.

### UFCS Resolution

When the interpreter encounters `x.f(args)`:

1. Check if `x` is a product type with a field named `f`. If so, access the field.
2. If the field is callable and `(args)` follows, call it.
3. Otherwise, look up `f` in the current scope as a free function. Rewrite the call to `f(x, args)`.
4. If none of the above, runtime error.

### Parenthesis Indexing

Lists, strings, and other indexable types use parenthesis syntax: `list(0)`, `scale(2)`. When a non-callable value appears in call position with an integer argument, it's an index operation:

```rust
fn call_value(callee: &Value, args: &[Value]) -> Result<Value, Error> {
    match callee {
        Value::Closure(c) => call_closure(c, args),
        Value::HostFn(f) => call_host_fn(f, args),
        Value::List(items) => {
            let index = args[0].as_int()?;
            Ok(items[index as usize].clone())
        }
        Value::Str(s) => {
            let index = args[0].as_int()?;
            Ok(Value::Str(Arc::from(&s[index..=index])))
        }
        _ => Err(Error::NotCallable(callee.type_name())),
    }
}
```

---

## 8.3 Hosting API

Clef is designed to be embedded. Two influences:

- **Lua:** Host creates and owns the interpreter state. Communication via a simple API. Host registers native functions. Multiple independent states can coexist.
- **Roc:** Host controls what capabilities are available. Clef scripts are pure computation — all I/O comes from host-provided functions.

### The Rust API (`clef-core`)

```rust
pub struct ClefState { /* internal */ }

impl ClefState {
    pub fn new() -> Self;                         // with standard library
    pub fn bare() -> Self;                        // no stdlib
    pub fn load_stdlib_music(&mut self);
    pub fn load_stdlib_seq(&mut self);
    pub fn load_stdlib_math(&mut self);
    pub fn load_stdlib_tuning(&mut self);
    pub fn eval(&mut self, source: &str) -> Result<Value, ClefError>;
    pub fn eval_file(&mut self, path: &Path) -> Result<Value, ClefError>;
    pub fn get_global(&self, name: &str) -> Option<Value>;
    pub fn set_global(&mut self, name: &str, value: Value);
    pub fn register_fn<F>(&mut self, name: &str, f: F)
    where F: Fn(&[Value]) -> Result<Value, ClefError> + 'static;
    pub fn realize(&mut self, music: &Value) -> Result<Vec<Event>, ClefError>;
}

/// The bridge type between Clef and host audio engines
#[derive(Clone, Debug)]
pub struct Event {
    pub time: BigRational,
    pub pitch_cents: f64,
    pub pitch_ratio: Option<BigRational>,
    pub duration: BigRational,
    pub velocity: f64,
    pub instrument: String,
}
```

### The C API (`clef-capi`)

A thin `extern "C"` wrapper enabling hosting from C, C++, Python, Go, Swift, and any language with C FFI.

```c
/* clef.h — Clef Embeddable Runtime C API */

#ifndef CLEF_H
#define CLEF_H

#include <stdint.h>
#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

/* Opaque handles */
typedef struct ClefState ClefState;
typedef struct ClefValue ClefValue;

/* Lifecycle */
ClefState* clef_new(void);
ClefState* clef_bare(void);
void       clef_free(ClefState* state);

/* Evaluation */
int         clef_eval(ClefState* state, const char* source, size_t len);
int         clef_eval_file(ClefState* state, const char* path);
const char* clef_last_error(ClefState* state);

/* Globals */
ClefValue*  clef_get_global(ClefState* state, const char* name);
void        clef_set_global(ClefState* state, const char* name, ClefValue* val);

/* Host functions */
typedef ClefValue* (*ClefHostFn)(ClefState*, int, const ClefValue* const*);
void clef_register_fn(ClefState* state, const char* name, ClefHostFn fn);

/* Value construction */
ClefValue*  clef_nil(void);
ClefValue*  clef_bool(int b);
ClefValue*  clef_int(int64_t n);
ClefValue*  clef_ratio(int64_t numer, int64_t denom);
ClefValue*  clef_float(double n);
ClefValue*  clef_string(const char* s, size_t len);

/* Value inspection */
typedef enum {
    CLEF_NIL, CLEF_BOOL, CLEF_INT, CLEF_RATIO, CLEF_FLOAT,
    CLEF_STRING, CLEF_LIST, CLEF_MUSIC, CLEF_EDO, CLEF_CENTS, CLEF_HZ,
    CLEF_CLOSURE, CLEF_TYPE_INSTANCE, CLEF_ENUM_VARIANT,
} ClefValueKind;

ClefValueKind clef_kind(const ClefValue* val);
int64_t       clef_as_int(const ClefValue* val);
double        clef_as_float(const ClefValue* val);
const char*   clef_as_string(const ClefValue* val, size_t* out_len);

/* Reference counting */
void clef_retain(ClefValue* val);
void clef_release(ClefValue* val);

/* Music */
ClefValue*  clef_realize(ClefState* state, ClefValue* music);
int         clef_event_count(const ClefValue* events);
double      clef_event_time(const ClefValue* events, int index);
double      clef_event_pitch_cents(const ClefValue* events, int index);
double      clef_event_duration(const ClefValue* events, int index);
double      clef_event_velocity(const ClefValue* events, int index);
const char* clef_event_instrument(const ClefValue* events, int index);

/* Selective stdlib loading */
void clef_load_stdlib_all(ClefState* state);
void clef_load_stdlib_math(ClefState* state);
void clef_load_stdlib_seq(ClefState* state);
void clef_load_stdlib_music(ClefState* state);
void clef_load_stdlib_tuning(ClefState* state);

#ifdef __cplusplus
}
#endif
#endif /* CLEF_H */
```

### C API Design Principles

- **Small and stable.** Every function is a compatibility promise. Start minimal; add convenience later.
- **Opaque pointers.** `ClefState*` and `ClefValue*` hide internals. The Rust implementation can change without breaking the ABI.
- **Explicit refcounting.** Host receives one reference from API calls. Must call `clef_release` when done, `clef_retain` to keep longer. Same model as Core Foundation / COM.
- **No exceptions across FFI.** Errors via return codes and `clef_last_error()`.
- **Thread safety rule.** A single `ClefState` is NOT thread-safe. For concurrent use, create multiple states. The audio thread reads realized event lists (plain data, `Arc`-shared) but never touches the `ClefState`.

---

## 8.4 WASM Target

### Architecture

```
Browser:
┌─────────────────────────────────────────────┐
│  JS/TS Host                                  │
│  ├─ Monaco Editor (code editing)             │
│  ├─ Web Audio API (real-time rendering)      │
│  ├─ Canvas/WebGL (visualization)             │
│  └─ calls into ↓ via wasm-bindgen            │
├─────────────────────────────────────────────┤
│  Clef Runtime (WASM)                         │
│  ├─ eval(source) → Value                     │
│  ├─ realize(music) → Event[]                 │
│  ├─ Rc-based (single-threaded, no atomics)   │
│  └─ target: < 1MB module                     │
└─────────────────────────────────────────────┘
```

### WASM-Specific Details

**`Rc` vs `Arc`:** WASM is single-threaded, so use `Rc<T>` instead of `Arc<T>` to avoid atomic overhead:

```rust
#[cfg(target_arch = "wasm32")]
type Shared<T> = std::rc::Rc<T>;

#[cfg(not(target_arch = "wasm32"))]
type Shared<T> = std::sync::Arc<T>;
```

**No filesystem.** WASM build excludes `eval_file` and all file I/O. Interaction is `eval(source_string)` + host functions.

**Module size target:** Under 1MB. If `num` crate bignums are too large, use 64-bit numerator/denominator with overflow detection for WASM, full bignums for native.

**JS bindings:**

```rust
#[wasm_bindgen]
pub struct ClefWasm { state: ClefState }

#[wasm_bindgen]
impl ClefWasm {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Self;
    pub fn eval(&mut self, source: &str) -> Result<JsValue, JsError>;
    pub fn realize_to_json(&mut self, global_name: &str) -> Result<String, JsError>;
}
```

---

## 8.5 Host Patterns

### Pattern 1: Browser Playground

User edits code, clicks Play. Events rendered via Web Audio, visualized on Canvas.

```
User code → ClefWasm.eval() → ClefWasm.realize_to_json()
                                         │
                          ┌──────────────┴──────────────┐
                          ▼                              ▼
                   Web Audio API                  Canvas / WebGL
                   (schedule notes)            (piano roll / tree viz)
```

### Pattern 2: DAW with Clef Scripting

Desktop DAW embeds Clef. Audio thread and Clef thread are separate.

```
UI Thread:                        Audio Thread:
  ClefState (owns interpreter)      reads Arc<Vec<Event>>
  on edit → eval → realize          schedules sample-accurately
  sends events via channel          runs synthesis (native)
                                    NEVER touches ClefState
```

### Pattern 3: Sandboxed Plugin System (Roc-style)

Host provides only curated primitives. Untrusted scripts cannot access filesystem, network, or anything the host doesn't explicitly provide.

```rust
let mut clef = ClefState::bare();  // no stdlib at all
clef.register_fn("note", host_note_fn);
clef.register_fn("seq", host_seq_fn);
// no file I/O, no network — scripts are sandboxed by construction
clef.eval(untrusted_user_script)?;
```

### Pattern 4: Embedding in Other Languages

Via the C API, Clef can be hosted from any language:

- **Python:** `ctypes.cdll.LoadLibrary("libclef.so")` → call `clef_new`, `clef_eval`, etc.
- **Go:** `cgo` with `#include "clef.h"`
- **Swift:** direct C interop
- **C++:** include `clef.h`, link against `libclef`
- **Node.js:** N-API wrapper or WASM module

---

## 8.6 Crate Structure

```
clef/
├── clef-core/            # core: scanner, parser, interpreter, values
│   ├── src/
│   │   ├── lib.rs
│   │   ├── scanner.rs
│   │   ├── token.rs
│   │   ├── parser.rs
│   │   ├── ast.rs
│   │   ├── interpreter.rs
│   │   ├── value.rs
│   │   ├── music.rs
│   │   ├── ratio.rs
│   │   ├── environment.rs
│   │   ├── error.rs
│   │   └── stdlib/
│   │       ├── mod.rs
│   │       ├── math.rs
│   │       ├── seq.rs
│   │       ├── music.rs
│   │       └── tuning.rs
│   └── Cargo.toml
│
├── clef-cli/             # REPL and file runner
│   ├── src/main.rs
│   └── Cargo.toml
│
├── clef-capi/            # C API (extern "C")
│   ├── src/lib.rs
│   ├── include/clef.h
│   └── Cargo.toml        # cdylib + staticlib
│
├── clef-wasm/            # WASM bindings
│   ├── src/lib.rs
│   └── Cargo.toml        # wasm32-unknown-unknown
│
└── Cargo.toml            # workspace
```

| Target | Crate | Output | Use |
|--------|-------|--------|-----|
| Rust library | `clef-core` | `libclef.rlib` | Rust hosts link directly |
| C shared lib | `clef-capi` | `libclef.so` / `.dll` / `.dylib` | Any language with C FFI |
| C static lib | `clef-capi` | `libclef.a` | Statically linked hosts |
| WASM | `clef-wasm` | `.wasm` + JS glue | Browser playground |
| CLI | `clef-cli` | `clef` binary | REPL, running .clef files |

---

## 8.7 Future Directions

- **Bytecode VM.** Compile to bytecode for ~10x speedup over tree-walk. Stack-based, `YIELD` instruction for generators.
- **Operator overloading.** `++` for `seq`, `&` for `par`. Requires traits/protocols.
- **Module system.** `import`/`export` across files.
- **Live coding.** Hot-swap definitions during playback.
- **Probability / randomness.** Weighted choice, Markov chains.
- **Spatial audio.** Pan, Ambisonics as `Control` modifiers.
- **Notation output.** LilyPond, MusicXML, SVG export.
- **Property-based testing.** Built-in framework for algebraic law verification.
- **Cycle collector.** Optional, host-invoked, for long-running sessions.
- **JIT via Cranelift.** Distant future — compile hot generator loops to native code.
