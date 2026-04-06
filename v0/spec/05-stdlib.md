# 5. Standard Library

> **Created:** 2026-04-05  
> **Last Updated:** 2026-04-06  
> **Changelog:**
> - 2026-04-05 — Initial draft
> - 2026-04-06 — Added metadata headers

## 5.1 Sequence Operations

All sequence operations are free functions, chainable via UFCS.

### Creation

```
fun range(start: Int, end: Int): Seq[Int]
fun repeat[T](value: T): Seq[T]           // infinite repetition
fun cycle[T](items: list[T]): Seq[T]      // infinite cycling
fun empty[T](): Seq[T]
```

### Transformation

```
fun map[T, U](seq: Seq[T], f: (T) => U): Seq[U]
fun flat_map[T, U](seq: Seq[T], f: (T) => Seq[U]): Seq[U]
fun filter[T](seq: Seq[T], f: (T) => bool): Seq[T]
fun flatten[T](seq: Seq[Seq[T]]): Seq[T]
fun zip[T, U](a: Seq[T], b: Seq[U]): Seq[(T, U)]
fun scan[T, U](seq: Seq[T], init: U, f: (U, T) => U): Seq[U]
fun chunk[T](seq: Seq[T], size: number): Seq[list[T]]
```

### Consumption

```
fun take[T](seq: Seq[T], n: number): Seq[T]
fun drop[T](seq: Seq[T], n: number): Seq[T]
fun take_while[T](seq: Seq[T], f: (T) => bool): Seq[T]
fun collect[T](seq: Seq[T]): list[T]         // forces evaluation
fun reduce[T](seq: Seq[T], f: (T, T) => T): T
fun fold[T, U](seq: Seq[T], init: U, f: (U, T) => U): U
fun find[T](seq: Seq[T], f: (T) => bool): T?
fun count[T](seq: Seq[T]): Int               // forces evaluation
fun any[T](seq: Seq[T], f: (T) => bool): bool
fun all[T](seq: Seq[T], f: (T) => bool): bool
```

### Reordering

```
fun reverse[T](seq: Seq[T]): Seq[T]          // requires finite seq
fun rotate[T](seq: Seq[T], n: number): Seq[T]
fun sort_by[T, U: Ord](seq: Seq[T], f: (T) => U): Seq[T]
```

## 5.2 List Operations

```
fun len[T](l: list[T]): Int
fun push[T](l: list[T], item: T): list[T]       // returns new list
fun concat[T](a: list[T], b: list[T]): list[T]
fun slice[T](l: list[T], start: Int, end: Int): list[T]
fun index_of[T](l: list[T], item: T): Int?
fun contains[T](l: list[T], item: T): bool
fun fill[T](n: Int, value: T): list[T]
```

## 5.3 Math Functions

```
fun abs(x: number): number
fun max(a: number, b: number): number
fun min(a: number, b: number): number
fun floor(x: Float): Int
fun ceil(x: Float): Int
fun round(x: Float): Int
fun sqrt(x: Float): Float
fun pow(base: number, exp: number): number
fun log(x: Float): Float
fun log2(x: Float): Float
fun sin(x: Float): Float
fun cos(x: Float): Float

// rational-specific
fun numer(r: Ratio): Int          // numerator
fun denom(r: Ratio): Int          // denominator
fun simplify(r: Ratio): Ratio     // reduce to lowest terms (auto on construction)
fun float(r: Ratio): Float        // explicit inexact conversion

// integer division (chainable via UFCS: 7.div(2) == 3)
fun div(a: Int, b: Int): Int      // truncating integer division
fun mod(a: Int, b: Int): Int      // remainder
```

## 5.4 Music Construction

```
// primitives — these construct Music enum variants
fun note(pitch: Interval, duration: Ratio): Music =
    Music.Note(pitch, duration, 80)

fun note(pitch: Interval, duration: Ratio, velocity: number): Music =
    Music.Note(pitch, duration, velocity)

fun rest(duration: Ratio): Music =
    Music.Rest(duration)

// composition (chainable via UFCS)
fun seq(a: Music, b: Music): Music = Music.Seq(a, b)
fun par(a: Music, b: Music): Music = Music.Par(a, b)

// transformations (chainable via UFCS)
fun transpose(m: Music, interval: Interval): Music =
    Music.Modify(Control.Transpose(interval), m)

fun invert(m: Music, axis: Interval): Music
fun retrograde(m: Music): Music
fun scale_tempo(m: Music, factor: Ratio): Music =
    Music.Modify(Control.Tempo(factor), m)

fun set_instrument(m: Music, name: string): Music =
    Music.Modify(Control.Instrument(name), m)

fun set_volume(m: Music, level: number): Music =
    Music.Modify(Control.Volume(level), m)

// queries
fun duration(m: Music): Ratio
fun pitches(m: Music): list[Interval]
fun count_notes(m: Music): Int
```

### How `transpose`, `invert`, and `retrograde` Work

`transpose` and `scale_tempo` wrap the Music tree in a `Modify` node — the transformation is applied lazily during `realize`. This makes them O(1) to apply regardless of tree size.

`invert` and `retrograde` are structural transformations that must walk the tree:

```
// invert reflects each pitch around an axis
fun invert(m: Music, axis: Interval): Music = match m {
    Note(p, d, v) => note(axis - (p - axis), d, v),
    Rest(d) => rest(d),
    Seq(a, b) => invert(a, axis).seq(invert(b, axis)),
    Par(a, b) => invert(a, axis).par(invert(b, axis)),
    Modify(ctrl, inner) => Music.Modify(ctrl, invert(inner, axis))
}

// retrograde reverses sequential order, leaves parallel untouched
fun retrograde(m: Music): Music = match m {
    Note(p, d, v) => m,                       // single note: unchanged
    Rest(d) => m,
    Seq(a, b) => retrograde(b).seq(retrograde(a)),  // swap and recurse
    Par(a, b) => retrograde(a).par(retrograde(b)),   // recurse both
    Modify(ctrl, inner) => Music.Modify(ctrl, retrograde(inner))
}
```

## 5.5 Tuning and Scales

```
// EDO construction — returns a list of all steps
fun edo(divisions: Int): list[Edo]
fun edo(divisions: Int, period: Ratio): list[Edo]    // non-octave EDOs (e.g., Bohlen-Pierce)

// conversion
fun cents(i: Interval): Cents
fun ratio(e: Edo): Ratio                  // nearest rational approximation
fun ratio(c: Cents): Ratio                // nearest rational approximation
fun freq(i: Interval, ref: Hz): Hz        // to frequency given reference

// scale utilities
fun nearest(scale: list[Interval], pitch: Interval): Interval
fun mode(scale: list[Interval], degree: Int): list[Interval]

// just intonation
fun limit(n: Int): list[Ratio]            // all n-limit ratios within an octave
fun odd_limit(n: Int): list[Ratio]        // all n-odd-limit ratios
```

## 5.6 Mixing and Voices

`mix` is a **library function**, not a language keyword. It combines multiple independent voices into a single `Music` tree. This section explains both how to use it and how it's implemented.

### Usage

```
let piece = mix([
    voice(bass_line),
    voice(chords),
    voice(arp).offset(1/4),
    voice(drone).tempo(5/4).volume(0.6),
])
```

### Voice Construction and Modifiers

A `Voice` wraps a `Seq[Note]` with playback metadata. All modifiers return new `Voice` values (immutable).

```
type Voice {
    events: Seq[Note],
    offset_beats: Ratio,
    tempo_factor: Ratio,
    instrument_name: string,
    volume_level: number,
}

// create a voice from a note sequence
fun voice(seq: Seq[Note]): Voice = Voice {
    events: seq,
    offset_beats: 0/1,
    tempo_factor: 1/1,
    instrument_name: "default",
    volume_level: 1.0,
}

// modifiers (UFCS-chainable, return new Voice)
fun offset(v: Voice, beats: Ratio): Voice =
    v.with(offset_beats: v.offset_beats + beats)

fun tempo(v: Voice, factor: Ratio): Voice =
    v.with(tempo_factor: v.tempo_factor * factor)

fun instrument(v: Voice, name: string): Voice =
    v.with(instrument_name: name)

fun volume(v: Voice, level: number): Voice =
    v.with(volume_level: v.volume_level * level)
```

### How `mix` Is Implemented

`mix` takes a list of voices and produces a `Music` tree by:

1. **Converting each voice's `Seq[Note]` into a `Music` value** — the notes in the sequence are chained with `.seq()` into a sequential `Music` tree.

2. **Applying voice modifiers as `Music.Modify` wrappers:**
   - `tempo_factor` becomes `Control.Tempo(factor)`
   - `instrument_name` becomes `Control.Instrument(name)`
   - `volume_level` becomes `Control.Volume(level)`
   - `offset_beats` is handled by prepending a `Rest` of the appropriate duration

3. **Combining all voices with `.par()`** — parallel composition means they play simultaneously.

Here is the implementation in Clef:

```
fun mix(voices: list[Voice]): Music {
    // convert each voice to a Music tree
    let music_voices = voices.map(v => {
        // step 1: convert Seq[Note] to sequential Music
        let music = v.events
            .map(n => note(n.pitch, n.duration, n.velocity))
            .reduce((a, b) => a.seq(b))

        // step 2: apply tempo scaling
        let temped = if v.tempo_factor != 1/1 {
            music.scale_tempo(v.tempo_factor)
        } else {
            music
        }

        // step 3: apply instrument
        let instrumented = if v.instrument_name != "default" {
            temped.set_instrument(v.instrument_name)
        } else {
            temped
        }

        // step 4: apply volume
        let volumed = if v.volume_level != 1.0 {
            instrumented.set_volume(v.volume_level)
        } else {
            instrumented
        }

        // step 5: prepend offset as a rest
        if v.offset_beats > 0/1 {
            rest(v.offset_beats).seq(volumed)
        } else {
            volumed
        }
    })

    // step 6: combine all voices in parallel
    music_voices.reduce((a, b) => a.par(b))
}
```

### Why `mix` Is a Library Function

Because `mix` is built from `voice`, `seq`, `par`, `rest`, and `Modify` — all standard library primitives — users can write their own mixing strategies. For example:

```
// interleave: alternate notes between voices instead of playing simultaneously
fun interleave(voices: list[Seq[Note]]): Seq[Note] {
    let zipped = voices.map(v => v.map(n => [n]))
    // ... custom interleaving logic
}

// round: start each voice at staggered intervals (like a musical round)
fun round(melody: Seq[Note], n_voices: Int, stagger: Ratio): Music {
    let voices = gen i in 0..n_voices {
        voice(melody).offset(stagger * i)
    }.collect
    mix(voices)
}

// layer: additive synthesis — same melody at multiple transpositions
fun layer(m: Music, intervals: list[Interval]): Music =
    intervals
        .map(i => m.transpose(i))
        .reduce((a, b) => a.par(b))
```

None of these require special language support. They compose from the same primitives that `mix` uses.

## 5.7 Realization and Rendering

### Event Type

`Event` is the concrete, scheduled representation of a note — the output of `realize`.

```
type Event {
    time: Ratio,            // beat position (absolute, from start)
    pitch: Interval,
    duration: Ratio,
    velocity: number,
    instrument: string,
}
```

### How `realize` Works

`realize` walks a `Music` tree and produces a time-sorted `Seq[Event]`. It maintains a **context** that tracks the current time position, active instrument, tempo scale, and volume.

```
type RealizeContext {
    time: Ratio,            // current beat position
    tempo: Ratio,           // cumulative tempo scaling
    instrument: string,
    volume: number,
}

fun realize(m: Music): Seq[Event] =
    realize_with(m, RealizeContext {
        time: 0/1,
        tempo: 1/1,
        instrument: "default",
        volume: 1.0,
    })

fun realize_with(m: Music, ctx: RealizeContext): Seq[Event] = match m {
    Note(p, d, v) => [Event {
        time: ctx.time,
        pitch: p,
        duration: d * ctx.tempo,
        velocity: v * ctx.volume,
        instrument: ctx.instrument,
    }],

    Rest(d) => [],    // rests produce no events (but advance time in Seq)

    Seq(a, b) => {
        let events_a = realize_with(a, ctx)
        let dur_a = duration(a) * ctx.tempo
        let ctx_b = ctx.with(time: ctx.time + dur_a)
        let events_b = realize_with(b, ctx_b)
        events_a.concat(events_b)
    },

    Par(a, b) => {
        // both start at the same time
        let events_a = realize_with(a, ctx)
        let events_b = realize_with(b, ctx)
        merge_sorted(events_a, events_b, (e) => e.time)
    },

    Modify(Tempo(f), inner) =>
        realize_with(inner, ctx.with(tempo: ctx.tempo * f)),

    Modify(Transpose(i), inner) => {
        // realize inner, then shift all pitches
        realize_with(inner, ctx).map(e => e.with(pitch: e.pitch + i))
    },

    Modify(Instrument(name), inner) =>
        realize_with(inner, ctx.with(instrument: name)),

    Modify(Volume(level), inner) =>
        realize_with(inner, ctx.with(volume: ctx.volume * level)),

    Modify(Custom(name, value), inner) =>
        realize_with(inner, ctx),   // custom controls ignored by default realize
}
```

### Rendering (Host FFI)

These functions bridge to the host audio engine. Their implementations are native (provided by the host via FFI), not written in Clef.

```
fun render(events: Seq[Event], bpm: number, ref: Hz, sample_rate: Int): Audio
fun export(audio: Audio, path: string): nil
fun to_midi(events: Seq[Event], path: string): nil
```

### Conversion Between Structural and Generative Layers

```
// Seq[Note] → Music: chain notes sequentially into a Music tree
fun to_music(seq: Seq[Note]): Music =
    seq.map(n => note(n.pitch, n.duration, n.velocity))
       .reduce((a, b) => a.seq(b))

// take a duration's worth of notes from a sequence
fun take_beats(seq: Seq[Note], beats: Ratio): Seq[Note] {
    mut remaining = beats
    seq.take_while(n => {
        let take = remaining > 0/1
        remaining = remaining - n.duration
        take
    })
}
```
