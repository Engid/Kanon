# Kanon

A programming language for writing music — especially microtonal music.

Intervals are exact ratios, not approximations. Tunings, scales, and structure are things you express directly. You write music as code, and Kanon turns it into events any audio engine can play.

```
let chord = mix([
    note(offset:0, 5/4, 1/1),
    note(offset:0, 3/2, 1/1)
])

// sends music data to host for playback
realize(chord)
```

Kanon is meant to be embedded — in a DAW, a web playground, a live-coding tool. It handles composition; the host handles sound. Early stages: the spec is taking shape, implementation is just getting started.

## The name

Named after the Pythagorean monochord (*kanōn*) — the instrument used to discover that music is math. It also echoes the musical *canon*. Kanon is also common Japanese name with musical meanings: 奏音 (*play + sound*) and 歌音 (*song + sound*).

Files use the `.kan` extension.

## Docs

- [Language overview](v0/spec/01-overview.md)
- [Spec index](v0/spec/00-index.md)
- [RFCs](v0/RFCs/)