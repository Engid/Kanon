(NOTE: this was copied out of the spec directory since it isn't language specific. spec should just define the language)

# 8. Implementation in Rust

## 8.1 Parsing Strategy

Use a hand-rolled recursive descent parser with Pratt parsing for expressions (see §6 for the full grammar and precedence table).

The scanner produces tokens; the parser consumes them to build an AST. A single-pass parser is sufficient for Kanon's grammar — there are no forward references that require multiple passes (type declarations can be used before they appear in the source via a pre-pass or lazy resolution).

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

Kanon v0.1 is a **tree-walk interpreter written in Rust**. This is the fastest path to a working implementation. A bytecode VM can be added later as an optimization (following the Crafting Interpreters progression: tree-walk first, bytecode second).

The runtime has these components:

```
┌─────────────────────────────────────────────────┐
│                  KanonState                      │
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

All Kanon values are represented as a Rust enum wrapped in `Arc` (atomic reference counting) for shared ownership:

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

**The cycle problem:** Reference cycles cause memory leaks with pure refcounting. For Kanon, this is manageable:

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

Kanon is designed to be embedded. Two influences:

- **Lua:** Host creates and owns the interpreter state. Communication via a simple API. Host registers native functions. Multiple independent states can coexist.
- **Roc:** Host controls what capabilities are available. Kanon scripts are pure computation — all I/O comes from host-provided functions.

### The Rust API (`kanon-core`)

```rust
pub struct KanonState { /* internal */ }

impl KanonState {
    pub fn new() -> Self;                         // with standard library
    pub fn bare() -> Self;                        // no stdlib
    pub fn load_stdlib_music(&mut self);
    pub fn load_stdlib_seq(&mut self);
    pub fn load_stdlib_math(&mut self);
    pub fn load_stdlib_tuning(&mut self);
    pub fn eval(&mut self, source: &str) -> Result<Value, KanonError>;
    pub fn eval_file(&mut self, path: &Path) -> Result<Value, KanonError>;
    pub fn get_global(&self, name: &str) -> Option<Value>;
    pub fn set_global(&mut self, name: &str, value: Value);
    pub fn register_fn<F>(&mut self, name: &str, f: F)
    where F: Fn(&[Value]) -> Result<Value, KanonError> + 'static;
    pub fn realize(&mut self, music: &Value) -> Result<Vec<Event>, KanonError>;
}

/// The bridge type between Kanon and host audio engines
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

### The C API (`kanon-capi`)

A thin `extern "C"` wrapper enabling hosting from C, C++, Python, Go, Swift, and any language with C FFI.

```c
/* kanon.h — Kanon Embeddable Runtime C API */

#ifndef KANON_H
#define KANON_H

#include <stdint.h>
#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

/* Opaque handles */
typedef struct KanonState KanonState;
typedef struct KanonValue KanonValue;

/* Lifecycle */
KanonState* kanon_new(void);
KanonState* kanon_bare(void);
void        kanon_free(KanonState* state);

/* Evaluation */
int         kanon_eval(KanonState* state, const char* source, size_t len);
int         kanon_eval_file(KanonState* state, const char* path);
const char* kanon_last_error(KanonState* state);

/* Globals */
KanonValue* kanon_get_global(KanonState* state, const char* name);
void        kanon_set_global(KanonState* state, const char* name, KanonValue* val);

/* Host functions */
typedef KanonValue* (*KanonHostFn)(KanonState*, int, const KanonValue* const*);
void kanon_register_fn(KanonState* state, const char* name, KanonHostFn fn);

/* Value construction */
KanonValue* kanon_nil(void);
KanonValue* kanon_bool(int b);
KanonValue* kanon_int(int64_t n);
KanonValue* kanon_ratio(int64_t numer, int64_t denom);
KanonValue* kanon_float(double n);
KanonValue* kanon_string(const char* s, size_t len);

/* Value inspection */
typedef enum {
    KANON_NIL, KANON_BOOL, KANON_INT, KANON_RATIO, KANON_FLOAT,
    KANON_STRING, KANON_LIST, KANON_MUSIC, KANON_EDO, KANON_CENTS, KANON_HZ,
    KANON_CLOSURE, KANON_TYPE_INSTANCE, KANON_ENUM_VARIANT,
} KanonValueKind;

KanonValueKind kanon_kind(const KanonValue* val);
int64_t        kanon_as_int(const KanonValue* val);
double         kanon_as_float(const KanonValue* val);
const char*    kanon_as_string(const KanonValue* val, size_t* out_len);

/* Reference counting */
void kanon_retain(KanonValue* val);
void kanon_release(KanonValue* val);

/* Music */
KanonValue* kanon_realize(KanonState* state, KanonValue* music);
int         kanon_event_count(const KanonValue* events);
double      kanon_event_time(const KanonValue* events, int index);
double      kanon_event_pitch_cents(const KanonValue* events, int index);
double      kanon_event_duration(const KanonValue* events, int index);
double      kanon_event_velocity(const KanonValue* events, int index);
const char* kanon_event_instrument(const KanonValue* events, int index);

/* Selective stdlib loading */
void kanon_load_stdlib_all(KanonState* state);
void kanon_load_stdlib_math(KanonState* state);
void kanon_load_stdlib_seq(KanonState* state);
void kanon_load_stdlib_music(KanonState* state);
void kanon_load_stdlib_tuning(KanonState* state);

#ifdef __cplusplus
}
#endif
#endif /* KANON_H */
```

### C API Design Principles

- **Small and stable.** Every function is a compatibility promise. Start minimal; add convenience later.
- **Opaque pointers.** `KanonState*` and `KanonValue*` hide internals. The Rust implementation can change without breaking the ABI.
- **Explicit refcounting.** Host receives one reference from API calls. Must call `kanon_release` when done, `kanon_retain` to keep longer. Same model as Core Foundation / COM.
- **No exceptions across FFI.** Errors via return codes and `kanon_last_error()`.
- **Thread safety rule.** A single `KanonState` is NOT thread-safe. For concurrent use, create multiple states. The audio thread reads realized event lists (plain data, `Arc`-shared) but never touches the `KanonState`.

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
│  Kanon Runtime (WASM)                        │
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
pub struct KanonWasm { state: KanonState }

#[wasm_bindgen]
impl KanonWasm {
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
User code → KanonWasm.eval() → KanonWasm.realize_to_json()
                                         │
                          ┌──────────────┴──────────────┐
                          ▼                              ▼
                   Web Audio API                  Canvas / WebGL
                   (schedule notes)            (piano roll / tree viz)
```

### Pattern 2: DAW with Kanon Scripting

Desktop DAW embeds Kanon. Audio thread and Kanon thread are separate.

```
UI Thread:                        Audio Thread:
    KanonState (owns interpreter)     reads Arc<Vec<Event>>
  on edit → eval → realize          schedules sample-accurately
  sends events via channel          runs synthesis (native)
                                    NEVER touches KanonState
```

### Pattern 3: Sandboxed Plugin System (Roc-style)

Host provides only curated primitives. Untrusted scripts cannot access filesystem, network, or anything the host doesn't explicitly provide.

```rust
let mut kanon = KanonState::bare();  // no stdlib at all
kanon.register_fn("note", host_note_fn);
kanon.register_fn("seq", host_seq_fn);
// no file I/O, no network — scripts are sandboxed by construction
kanon.eval(untrusted_user_script)?;
```

### Pattern 4: Embedding in Other Languages

Via the C API, Kanon can be hosted from any language:

- **Python:** `ctypes.cdll.LoadLibrary("libkanon.so")` → call `kanon_new`, `kanon_eval`, etc.
- **Go:** `cgo` with `#include "kanon.h"`
- **Swift:** direct C interop
- **C++:** include `kanon.h`, link against `libkanon`
- **Node.js:** N-API wrapper or WASM module

---

## 8.6 Crate Structure

```
kanon/
├── kanon-core/           # core: scanner, parser, interpreter, values
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
├── kanon-cli/            # REPL and file runner
│   ├── src/main.rs
│   └── Cargo.toml
│
├── kanon-capi/           # C API (extern "C")
│   ├── src/lib.rs
│   ├── include/kanon.h
│   └── Cargo.toml        # cdylib + staticlib
│
├── kanon-wasm/           # WASM bindings
│   ├── src/lib.rs
│   └── Cargo.toml        # wasm32-unknown-unknown
│
└── Cargo.toml            # workspace
```

| Target | Crate | Output | Use |
|--------|-------|--------|-----|
| Rust library | `kanon-core` | `libkanon.rlib` | Rust hosts link directly |
| C shared lib | `kanon-capi` | `libkanon.so` / `.dll` / `.dylib` | Any language with C FFI |
| C static lib | `kanon-capi` | `libkanon.a` | Statically linked hosts |
| WASM | `kanon-wasm` | `.wasm` + JS glue | Browser playground |
| CLI | `kanon-cli` | `kanon` binary | REPL, running .kan files |

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
