# v0 — TypeScript Prototype

**Status:** Active  
**Started:** 2026-04-05  
**Goal:** A working Kanon interpreter in TypeScript that runs in CLI (Bun) and eventually the browser. The purpose is to solidify the language spec through real usage, not to ship production software.

---

## Guiding Principles

1. **Mirror the spec's host/guest architecture from day one.** The runtime is pure computation. All I/O flows through a `KanonHost` interface.
2. **`core/` is platform-free.** Nothing in `core/` imports from `cli/`, `web/`, Node, Bun, or browser APIs. If it compiles, it works everywhere.
3. **Single package, directory boundaries.** No monorepo/workspace overhead. Directories are the module boundary. Split into packages only when there's a reason.
4. **Three-layer output model.** `parse()` → AST, `eval()` → values/Music trees, `realize()` → Event[]. Each stage is independently useful and testable.
5. **Tests verify the spec.** Tests are written against spec behavior, not implementation details. They serve as executable documentation of what Kanon *is*. Keep them modular so sections can be thrown away when the spec changes.
6. **Errors are first-class from the start.** Every pipeline stage produces structured errors with source locations. A language prototype is useless if you can't tell why your code broke.

---

## Testing Philosophy

Tests should be:

- **Spec-shaped, not implementation-shaped.** Test "this Kanon source produces this result" rather than "the parser calls this internal method." When the implementation changes, the spec-level tests survive.
- **Grouped by language feature, not by implementation file.** A test file like `rationals.test.ts` tests rational literals across scanner, parser, and interpreter — because that's how a spec reader thinks about it. A broken rational is a broken rational regardless of which pipeline stage failed.
- **Easy to gut and rewrite.** Each test file is self-contained. No shared test fixtures that create coupling. Helpers are small, local, and duplicated rather than shared. Deleting a test file should never break another test file.
- **Table-driven where possible.** Express expected behavior as data (input/output pairs), not procedural assertions. When the spec changes a behavior, you update one row in a table, not rewrite a test function.

```typescript
// GOOD: table-driven, spec-shaped, easy to update
const cases = [
  ["3/2",    { type: "ratio", numer: 3n, denom: 2n }],
  ["5/4",    { type: "ratio", numer: 5n, denom: 4n }],
  ["22/7",   { type: "ratio", numer: 22n, denom: 7n }],
  ["6/4",    { type: "ratio", numer: 3n, denom: 2n }],  // auto-simplified
] as const

for (const [input, expected] of cases) {
  test(`rational literal: ${input}`, () => {
    const result = evalExpr(input)
    expect(result).toMatchObject(expected)
  })
}
```

### Test directory structure

```
test/
  scan/                    # scanner-level tests
    literals.test.ts       # numeric literals, strings, unit suffixes
    keywords.test.ts       # keyword recognition, identifiers
    operators.test.ts      # all operator tokens
    comments.test.ts       # single-line, multi-line
    errors.test.ts         # unterminated strings, bad escapes, etc.
  parse/                   # parser-level tests (source → AST shape)
    expressions.test.ts    # arithmetic, comparison, logical, pipe, UFCS
    declarations.test.ts   # let, mut, fun, type, enum
    control-flow.test.ts   # if/else, for, while, match
    generators.test.ts     # gen expressions
    precedence.test.ts     # Pratt parser precedence edge cases
    errors.test.ts         # parse error recovery and messages
  eval/                    # interpreter-level tests (source → KanonValue)
    arithmetic.test.ts     # numeric tower, exactness, auto-simplify
    bindings.test.ts       # let, mut, scoping, shadowing
    functions.test.ts      # fun decl, lambdas, closures, recursion
    control-flow.test.ts   # if/else as expression, for, while, match
    types.test.ts          # product types, enums, .with(), construction
    generators.test.ts     # gen, yield, lazy evaluation
    ufcs.test.ts           # UFCS resolution, chaining
    errors.test.ts         # runtime error messages and locations
  stdlib/                  # stdlib function tests
    math.test.ts
    seq.test.ts
    music.test.ts
    tuning.test.ts
  music/                   # end-to-end music pipeline tests
    realize.test.ts        # Music tree → Event[] correctness
    laws.test.ts           # algebraic laws from spec §3.5
    examples.test.ts       # full programs from spec §7
  helpers.ts               # minimal shared helpers (evalExpr, parseExpr, etc.)
```

### The `helpers.ts` contract

Keep this tiny. It's the only shared file. It provides convenience wrappers so tests don't repeat boilerplate:

```typescript
// test/helpers.ts — keep this small and stable
export function scan(source: string): Token[]
export function parse(source: string): Ast.Program
export function evalSource(source: string): KanonValue      // full program
export function evalExpr(source: string): KanonValue        // single expression
export function realizeSource(source: string): Event[]      // eval + realize
export function testHost(): KanonHost                       // stub host that captures print() calls
```

If a test file needs something not in `helpers.ts`, it defines it locally. No creeping shared state.

---

## Phase 0: Project Scaffold

**Goal:** A buildable, testable TypeScript project with the architectural skeleton in place. No language logic yet — just the shape of the codebase and the contracts between layers.

### Deliverables

```
v0/
  src/
    core/
      scanner.ts           # exports scan(): Token[]  (stub: returns [])
      parser.ts            # exports parse(): Ast.Program  (stub: returns empty program)
      ast.ts               # AST node type definitions
      values.ts            # KanonValue discriminated union
      interpreter.ts       # exports eval(): KanonValue  (stub: returns Nil)
      environment.ts       # Environment class (scope chain)
      errors.ts            # KanonError type, error construction helpers
      stdlib/              # empty dir, populated in later phases
    host.ts                # KanonHost interface
    runtime.ts             # KanonRuntime class wiring core + host
    cli/
      main.ts              # entry point: kanon run / eval / ast
  test/
    helpers.ts             # test utilities
    smoke.test.ts          # "it compiles and runs" sanity check
  package.json
  tsconfig.json
```

### Key type definitions to establish

**`KanonHost`** — the boundary contract:
```typescript
interface KanonHost {
  print(value: string): void
}
```

**`KanonError`** — structured errors from any pipeline stage:
```typescript
interface KanonError {
  message: string
  line: number
  column: number
  phase: 'scan' | 'parse' | 'runtime'
}
```

**`KanonValue`** — the runtime value discriminated union (initially just the skeleton, filled in across phases):
```typescript
type KanonValue =
  | { type: "nil" }
  | { type: "bool"; value: boolean }
  | { type: "int"; value: bigint }
  | { type: "float"; value: number }
  | { type: "ratio"; numer: bigint; denom: bigint }
  | { type: "string"; value: string }
  // ... expanded in later phases
```

**`KanonRuntime`** — the pipeline orchestrator:
```typescript
class KanonRuntime {
  constructor(host: KanonHost)
  scan(source: string): Token[]
  parse(source: string): Ast.Program
  eval(source: string): KanonValue
  realize(music: KanonValue): Event[]
}
```

### Verification

- [ ] `bun test` passes (smoke test: runtime can be constructed, stubs return expected defaults)
- [ ] `bun run src/cli/main.ts` runs without error
- [ ] `core/` contains zero imports from `cli/`, `web/`, or platform APIs
- [ ] TypeScript compiles with strict mode, no errors

---

## Phase 1: Scanner

**Goal:** Tokenize all Kanon source forms defined in spec §2. The scanner takes a source string and produces a flat array of tokens with source locations.

### What to implement

- Single-line comments (`//`) and multi-line comments (`/* */`)
- Identifiers and all keywords from §2.3: `fun let mut type enum match if else for in gen yield return true false nil import as where and or not`
- Integer literals: decimal, hex (`0x`), binary (`0b`), octal (`0o`), underscore separators
- Float literals: with decimal point, with exponent (`e`/`E`)
- String literals with `${}` interpolation markers (scan as segments, don't evaluate)
- All operators from §2.6: `+ - * / % == != < <= > >= = |> .. ..=`
- Delimiters: `( ) { } [ ] , : . =>`
- Special tokens that the parser will use for compound literals: `\` (for EDO), `i` (imaginary suffix), `hz`, `c` (unit suffixes)
- Source location tracking: every token carries `line`, `column`, `start`, `end`

### Key scanner decisions

- **`/` is always a single token.** The parser decides if `INT / INT` is a ratio literal. The scanner doesn't need to know.
- **`\` is always a single token.** Same — parser resolves `INT \ INT` as EDO.
- **`i`, `hz`, `c` after number literals**: scan as separate identifier tokens. The parser or a post-scan pass recognizes `<number><ident:i|hz|c>` as a unit-suffixed literal. This keeps the scanner simple.
- **String interpolation**: `"hello ${name} world"` scans as multiple tokens: `STRING_START "hello "`, `INTERPOLATION_START`, `IDENT name`, `INTERPOLATION_END`, `STRING_END " world"`. This lets the parser handle the expression inside `${}` normally.

### Token type enum

```typescript
enum TokenType {
  // Literals
  INT, FLOAT, STRING, STRING_INTERP_START, STRING_INTERP_END,

  // Identifiers & keywords
  IDENT,
  FUN, LET, MUT, TYPE, ENUM, MATCH, IF, ELSE, FOR, IN,
  GEN, YIELD, RETURN, TRUE, FALSE, NIL,
  IMPORT, AS, WHERE, AND, OR, NOT, PRIVATE,

  // Operators
  PLUS, MINUS, STAR, SLASH, PERCENT,
  EQ, EQ_EQ, BANG_EQ,
  LT, LT_EQ, GT, GT_EQ,
  PIPE_GT,        // |>
  DOT_DOT,        // ..
  DOT_DOT_EQ,     // ..=
  FAT_ARROW,      // =>
  BACKSLASH,      // \  (EDO separator)

  // Delimiters
  LPAREN, RPAREN, LBRACE, RBRACE, LBRACKET, RBRACKET,
  COMMA, COLON, DOT,

  // Special
  EOF,
  ERROR,
}
```

### Verification

- [ ] **Keyword recognition**: all 21 keywords scan to their specific token types; identifiers that *look like* keywords but aren't (e.g., `funny`, `lettuce`) scan as IDENT
- [ ] **Integer literals**: decimal, hex, binary, octal, underscore separators all produce INT tokens with correct values
- [ ] **Float literals**: decimal point, exponent notation produce FLOAT tokens
- [ ] **String literals**: plain strings, escape sequences (`\n`, `\t`, `\\`, `\"`), interpolation segments
- [ ] **Operators**: every operator from §2.6 produces the correct token, including multi-char operators (`==`, `!=`, `|>`, `..`, `..=`, `=>`)
- [ ] **Comments**: single-line and multi-line comments are skipped, do not produce tokens, and nested `/* */` is handled (or explicitly disallowed)
- [ ] **Source locations**: spot-check that tokens carry correct line/column
- [ ] **Error tokens**: unterminated strings, unknown characters produce ERROR tokens with descriptive messages (not crashes)

---

## Phase 2: AST + Parser

**Goal:** Parse all Kanon syntax from spec §4 and §6 into a typed AST. The parser is a hand-rolled recursive descent with Pratt parsing for expressions, following the precedence table in §6.2.

### What to implement

**Declarations:**
- `let` / `mut` variable bindings (with optional type annotation)
- `fun` declarations (single-expression `= expr` and block `{ body }` forms)
- `type` declarations (product types with `{ fields }` and aliases with `= type`)
- `enum` declarations (variants with optional fields)

**Statements:**
- Expression statements
- `for` loops (`for x in expr { body }`)
- `while` loops
- `return`

**Expressions (Pratt parser, ascending precedence per §6.2):**
- Assignment: `x = expr`
- Pipe: `expr |> expr`
- Logical: `and`, `or`, `not`
- Equality: `==`, `!=`
- Comparison: `<`, `<=`, `>`, `>=`
- Addition: `+`, `-`
- Multiplication: `*`, `/`, `%`
- Unary: `-expr`, `not expr`
- Postfix: calls `f(args)`, member access `x.y`, imaginary suffix `i`
- Primary: literals, identifiers, `(expr)`, list literals `[a, b, c]`
- Lambda: `(params) => expr` and `(params) => { body }`
- Generator: `gen x in expr [where cond] expr|{body}`
- `if`/`else` (expression — returns a value)
- `match` (expression with arms, guards, nested patterns)
- Type constructor: `Name { field: value, ... }`
- Range: `start..end`, `start..=end`

**Compound literal recognition (in parser, not scanner):**
- `INT / INT` → `RatioLiteral` node (not `BinaryDiv`)
- `INT \ INT` → `EdoLiteral` node
- `<numeric> i` → `ImaginaryLiteral` node
- `<float> . hz` or `<float> . c` → handled via UFCS (no special AST node needed, or: recognize as `UnitLiteral` for better error messages)

### AST node design

Use discriminated unions. Each node carries a `span` for error reporting:

```typescript
interface Span { start: number; end: number; line: number; column: number }

type Expr =
  | { kind: "int_literal"; value: bigint; span: Span }
  | { kind: "float_literal"; value: number; span: Span }
  | { kind: "ratio_literal"; numer: bigint; denom: bigint; span: Span }
  | { kind: "edo_literal"; step: bigint; divisions: bigint; span: Span }
  | { kind: "string_literal"; parts: StringPart[]; span: Span }
  | { kind: "bool_literal"; value: boolean; span: Span }
  | { kind: "nil_literal"; span: Span }
  | { kind: "ident"; name: string; span: Span }
  | { kind: "binary"; op: BinaryOp; left: Expr; right: Expr; span: Span }
  | { kind: "unary"; op: UnaryOp; operand: Expr; span: Span }
  | { kind: "call"; callee: Expr; args: Arg[]; span: Span }
  | { kind: "member"; object: Expr; name: string; span: Span }
  | { kind: "lambda"; params: Param[]; body: Expr | Stmt[]; span: Span }
  | { kind: "if"; condition: Expr; then: Stmt[]; else_: Stmt[] | Expr | null; span: Span }
  | { kind: "match"; subject: Expr; arms: MatchArm[]; span: Span }
  | { kind: "gen"; name: string; iter: Expr; where_: Expr | null; body: Expr | Stmt[]; span: Span }
  | { kind: "list_literal"; elements: Expr[]; span: Span }
  | { kind: "type_constructor"; name: string; fields: FieldInit[]; span: Span }
  | { kind: "range"; start: Expr; end: Expr; inclusive: boolean; span: Span }
  | { kind: "pipe"; left: Expr; right: Expr; span: Span }
  | { kind: "imaginary"; expr: Expr; span: Span }
  | { kind: "assign"; name: string; value: Expr; span: Span }
  // ...
```

### Verification

- [ ] **Precedence**: `1 + 2 * 3` parses as `1 + (2 * 3)`, `a or b and c` as `a or (b and c)`, etc. — test every level of the Pratt table from §6.2
- [ ] **Rational literals**: `3/2` → `RatioLiteral(3, 2)`, not `BinaryDiv(3, 2)`
- [ ] **EDO literals**: `7\12` → `EdoLiteral(7, 12)`
- [ ] **Compound expressions**: `3/2 + 1/2i` parses correctly (ratio + imaginary)
- [ ] **Lambda parsing**: `(x) => x * 2`, `x => x * 2`, `(x, y) => { ... }`
- [ ] **Generator parsing**: `gen n in 1.. n * 2`, `gen n in 0..12 where n % 2 == 0 { yield n }`
- [ ] **Match**: arms with patterns, guards, block bodies, nested patterns
- [ ] **If/else as expression**: `let x = if cond { a } else { b }`
- [ ] **Type declarations**: `type Note { pitch: Interval, duration: Ratio }`, `type Scale = list[Interval]`
- [ ] **Enum declarations**: `enum Music { Note(...), Rest(...), Seq(...), Par(...), Modify(...) }`
- [ ] **Fun declarations**: single-expression (`= expr`) and block forms, with generics
- [ ] **UFCS chains**: `x.f(a).g(b).h(c)` parses as nested member+call
- [ ] **Parse errors**: missing closing brace, unexpected token, etc. produce `KanonError` with line/column, not crashes
- [ ] **Round-trip spot checks**: parse a few §7 examples and verify the AST shape makes sense (doesn't need to be exhaustive yet — just catch gross structural errors)

---

## Phase 3: Interpreter + Values

**Goal:** A working tree-walk interpreter that evaluates Kanon programs into `KanonValue` results. This is the core of the language — after this phase, Kanon programs actually *run*.

### What to implement

**Value system (expand `KanonValue`):**
- `nil`, `bool`, `int` (bigint), `float` (number), `ratio` (exact rational with GCD reduction), `string`
- `list` (immutable array of KanonValue)
- `closure` (captured environment + params + body)
- `host_fn` (TypeScript function registered by the host)
- `type_instance` (product type value with named fields)
- `enum_variant` (tagged union value)
- `edo` (step + divisions), `cents` (float), `hz` (float)

**Ratio arithmetic:**
- Implement a lightweight `Ratio` representation (two `bigint`s, auto-GCD on construction)
- Exactness rules from §3.1: Int op Int → Int/Ratio, Ratio op Ratio → Ratio, anything op Float → Float

**Environment / scope chain:**
- Lexical scoping: block `{}` creates a new scope
- `let` bindings are immutable, `mut` bindings allow reassignment
- Function bodies get their own scope, closures capture the defining environment
- UFCS resolution: `x.f(args)` → check field, then look up `f` in scope and rewrite to `f(x, args)`

**Interpreter eval (tree-walk):**
- Literal evaluation (all forms: int, float, ratio, edo, string, bool, nil, imaginary, list)
- Binary and unary operators (with numeric tower promotion rules)
- Variable lookup and assignment
- Function declaration and calls (with default parameters)
- Lambda creation and calls
- `if`/`else` as expression
- `for` and `while` loops
- `match` expression with pattern matching (constructor patterns, binding, wildcard, literal, guards)
- Type and enum declarations (register constructors in environment)
- Product type construction, field access, `.with()` copy-modification
- Enum variant construction
- `return` statement (early return from functions)
- Parenthesis indexing: `list(0)`, `string(2)` — when callee is a list/string, treat as index
- Pipe operator: `x |> f(y)` → `f(x, y)`
- String interpolation evaluation

**Generators (basic):**
- `gen x in iterable expr` → lazy Seq value
- `gen x in iterable where cond { yield expr }` — filtered/multi-yield
- Seq values are consumed lazily via stdlib functions (take, map, etc.)

### Verification

- [ ] **Numeric tower**: `1 + 2` → `Int(3)`, `3/2 + 1/2` → `Ratio(2, 1)`, `1 + 1.0` → `Float(2.0)`, `6/4` auto-simplifies to `Ratio(3, 2)`
- [ ] **Bindings**: `let x = 5; x` → `5`, `mut x = 1; x = 2; x` → `2`, reassigning `let` binding → error
- [ ] **Scoping**: inner blocks shadow outer bindings, closures capture correctly
- [ ] **Functions**: `fun f(x) = x * 2; f(3)` → `6`, recursive functions work, default params work
- [ ] **Lambdas**: `let f = (x) => x + 1; f(5)` → `6`, closures capture environment
- [ ] **If/else**: `if true { 1 } else { 2 }` → `1`, works as expression (assigned to variable)
- [ ] **Match**: basic pattern matching on enum variants, wildcard, guards, nested patterns
- [ ] **Product types**: construct, field access, `.with()` produces new value with changed field
- [ ] **Enums**: construct variant, match on variants
- [ ] **Lists**: `[1, 2, 3]` constructs, `list(0)` indexes, out-of-bounds → error
- [ ] **UFCS**: `fun double(x: Int) = x * 2; 5.double()` → `10`
- [ ] **Pipe**: `5 |> double` → `10`
- [ ] **String interpolation**: `let x = 42; "value: ${x}"` → `"value: 42"`
- [ ] **Generators**: `gen n in [1,2,3] n * 2` produces lazy seq that yields `[2, 4, 6]`
- [ ] **Error messages**: accessing undefined variable, type mismatch in operator, wrong arg count — all produce `KanonError` with source location
- [ ] **Spec §7.1** (hello world melody) runs: builds a list of edo values, maps them, reduces — produces a Music value (realization tested in Phase 4)

---

## Phase 4: Stdlib + Music + Realize

**Goal:** Implement the standard library functions from spec §5, the `Music` algebraic type, and `realize()` to produce `Event[]`. After this phase, Kanon programs produce playable music.

### What to implement

**Music types (add to `KanonValue`):**
- `music` variant: `Note`, `Rest`, `Seq`, `Par`, `Modify` — recursive tree
- `control` variant: `Tempo`, `Transpose`, `Instrument`, `Volume`, `Custom`
- `event` type: `{ time, pitch, duration, velocity, instrument }`

**Stdlib — Sequence ops (§5.1):**
- `map`, `filter`, `flat_map`, `flatten`, `zip`, `scan`, `chunk`
- `take`, `drop`, `take_while`, `collect`, `reduce`, `fold`
- `find`, `count`, `any`, `all`
- `reverse`, `rotate`, `sort_by`
- `range`, `repeat`, `cycle`, `empty`

**Stdlib — List ops (§5.2):**
- `len`, `push`, `concat`, `slice`, `index_of`, `contains`, `fill`

**Stdlib — Math (§5.3):**
- `abs`, `max`, `min`, `floor`, `ceil`, `round`, `sqrt`, `pow`, `log`, `log2`, `sin`, `cos`
- `numer`, `denom`, `simplify`, `float`
- `div`, `mod` (integer division)

**Stdlib — Music construction (§5.4):**
- `note`, `rest`, `seq`, `par`
- `transpose`, `invert`, `retrograde`, `scale_tempo`
- `set_instrument`, `set_volume`
- `duration`, `pitches`, `count_notes`

**Stdlib — Tuning (§5.5):**
- `cents`, `ratio` (conversion), `freq`
- `edo`, `nearest`, `mode`
- `limit`, `odd_limit`

**Stdlib — Mixing (§5.6):**
- `voice`, `offset`, `tempo`, `instrument`, `volume`
- `mix`

**`realize()` — Music tree → Event list (§5.7):**
- Walk the Music tree with a `RealizeContext` tracking time, tempo, instrument, volume
- `Note` → emit Event, `Rest` → advance time, `Seq` → recurse sequentially, `Par` → recurse in parallel (same start time), `Modify` → update context
- Output: sorted `Event[]`

**Stdlib registration pattern:**
Each stdlib module exports a `register(env: Environment)` function that binds its functions into the environment. The runtime calls these during construction:

```typescript
// core/stdlib/math.ts
export function register(env: Environment) {
  env.defineNative("abs", (args) => { ... })
  env.defineNative("max", (args) => { ... })
  // ...
}
```

### Verification

- [ ] **Seq ops**: `[1,2,3].map(x => x * 2).collect` → `[2, 4, 6]`, `[1,2,3,4].filter(x => x > 2).collect` → `[3, 4]`, `take`, `drop`, `reduce`, `fold` all work
- [ ] **Music construction**: `note(0\12, 1/4)` produces Music.Note, `.seq()` and `.par()` produce correct tree shapes
- [ ] **Music transformations**: `transpose`, `invert`, `retrograde` produce expected tree shapes, verified against §3.5 algebraic laws
- [ ] **Algebraic laws (§3.5)**: `a.seq(b).seq(c)` == `a.seq(b.seq(c))`, `a.par(b)` == `b.par(a)`, transpose distributes, retrograde is involution — these are property tests
- [ ] **realize()**: simple note sequence produces correct times, pitches, durations; parallel composition produces simultaneous events; Modify(Tempo) scales durations correctly
- [ ] **Spec §7.1** fully works — produces realized Event[] with correct times and pitches
- [ ] **Spec §7.2** (just intonation chord) — parallel notes at same time
- [ ] **Spec §7.3** (generative arpeggio) — gen + cycle + take
- [ ] **Spec §7.5** (structural transformations) — transpose, invert, retrograde
- [ ] **Spec §7.7** (pattern matching on Music) — `count_notes`, `all_pitches`
- [ ] **mix()** — multiple voices combine correctly with offsets, tempo scaling, instrument assignment

---

## Phase 5: CLI + REPL

**Goal:** A usable command-line interface for running Kanon programs, evaluating expressions, dumping ASTs, and interactive exploration.

### What to implement

**CLI commands (via Bun):**
- `kanon run <file.kan>` — evaluate a file, print the final value
- `kanon eval "<expr>"` — evaluate a single expression, print the result
- `kanon ast <file.kan>` — parse a file, pretty-print the AST as a tree
- `kanon realize <file.kan>` — evaluate + realize, print events as JSON/table

**REPL:**
- Interactive prompt, evaluate line-by-line
- Environment persists across lines (define a function, use it later)
- Print values with readable formatting (ratios as `3/2`, edo as `7\12`, Music trees as indented structure)
- Errors display with source location and the offending line

**Value pretty-printer:**
- Ratios: `3/2`
- Edo: `7\12`
- Music trees: indented structural view
- Events: table format (time | pitch | duration | velocity | instrument)
- Lists: `[1, 2, 3]` with truncation for long lists

### Verification

- [ ] `kanon run` on a file containing spec §7.1 produces output without error
- [ ] `kanon eval "3/2 + 5/4"` prints `11/4`
- [ ] `kanon ast` on a small program prints a readable tree
- [ ] REPL: multiline definitions work, previous bindings persist
- [ ] Errors display with line/column pointing at the problem

---

## Phase 6: Browser Host (Future)

**Goal:** A browser-based playground where users write Kanon code, hear it via Web Audio, and see visualizations.

This phase is deliberately unspecified — by the time we get here, the language will have evolved enough that detailed planning now would be wasted. The architecture supports it: `KanonRuntime` with a browser `KanonHost` that routes `print()` to a UI panel, and `Event[]` to Web Audio scheduling.

**Rough shape:**
- Code editor (Monaco or CodeMirror)
- "Run" button → `runtime.eval()` + `runtime.realize()` → schedule events via Web Audio
- Piano roll visualization of `Event[]`
- AST tree view (collapsible)
- Music tree view (the `Music` algebraic type as a visual tree)

Not planned in detail until Phases 0–5 are solid.

---

## Open Decisions to Resolve During Implementation

These are spec ambiguities or choices that will become concrete as we build:

1. **Indexing syntax**: `list(0)` vs `list[0]` — spec says `()`, but §10 flags the asymmetry with `[]` list literals. We'll go with `()` per spec and see if it feels wrong in practice.
2. **Error handling**: `Result[T, E]` and `?` operator — defer for v0 prototype. Use panics (thrown exceptions in TS) for errors. Add Result when we need it.
3. **Interfaces**: Defer entirely per spec recommendation. Free functions + UFCS cover v0.
4. **String interpolation depth**: Do we support arbitrary expressions in `${}`? (e.g., `"${if x > 0 { "pos" } else { "neg" }}"`) — start with simple expressions and identifiers, expand if needed.
5. **Module system**: Defer `import` for v0. Single-file programs only. Add when we need multi-file.
6. **`hz` / `c` unit suffixes**: Are they special syntax or UFCS calls to conversion functions? Spec §6.3 suggests UFCS. Let's try that and see if the parser handles it cleanly.
7. **Spread syntax in type constructors**: `Note { ...n, pitch: 9\12 }` — spec mentions it in the grammar but it overlaps with `.with()`. Pick one for v0.
