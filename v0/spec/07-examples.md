# 7. Example Programs

> **Created:** 2026-04-05  
> **Last Updated:** 2026-04-06  
> **Changelog:**
> - 2026-04-05 — Initial draft
> - 2026-04-06 — Added metadata headers

## 7.1 Hello World — A Simple Melody

```
let scale = [0\12, 2\12, 4\12, 5\12, 7\12, 9\12, 11\12]

let melody = scale
    .map(p => note(p, 1/4))
    .reduce((a, b) => a.seq(b))

melody.realize
    .render(bpm: 120, ref: 262.0hz, sample_rate: 48000)
    .export("hello.wav")
```

## 7.2 Just Intonation Chord

```
let root = 1/1
let major_chord = [root, 5/4, 3/2]

let chord = major_chord
    .map(p => note(p, 1/1))
    .reduce((a, b) => a.par(b))

chord.realize
    .render(bpm: 60, ref: 220.0hz, sample_rate: 48000)
    .export("just_chord.wav")
```

## 7.3 Generative Arpeggio in 19-EDO

```
let edo19_major = [0\19, 3\19, 6\19, 8\19, 11\19, 14\19, 17\19]

let arp = gen p in cycle(edo19_major) note(p, 1/8)

let four_bars = arp.take(32).to_music

four_bars.realize
    .render(bpm: 140, ref: 262.0hz, sample_rate: 48000)
    .export("arp_19edo.wav")
```

## 7.4 Polymetric Composition in 31-EDO

```
let major = [0\31, 5\31, 10\31, 13\31, 18\31, 23\31, 28\31]

fun arpeggiate(chord: list[Interval], dur: Ratio): Seq[Note] =
    gen p in cycle(chord) note(p, dur)

let chords = [
    [major(0), major(2), major(4)],
    [major(5), major(0), major(2)],
    [major(4), major(6), major(1)],
    [major(3), major(5), major(0)],
]

let bass = gen c in cycle(chords) note(c[0], 1/1)
let pads = gen c in cycle(chords) {
    c.map(p => note(p, 1/2)).reduce((a, b) => a.par(b))
}
let arp = gen c in cycle(chords) {
    arpeggiate(c, 1/8).take(8)
}.flatten

let piece = mix([
    voice(bass),
    voice(pads),
    voice(arp),
    voice(
        gen p in cycle(major.reverse) note(p + 18\31, 1/4)
    ).offset(1/4),
])

piece.realize
    .take_beats(32 * 4/1)
    .render(bpm: 120, ref: 262.0hz, sample_rate: 48000)
    .export("poly_31edo.wav")
```

## 7.5 Structural Composition with Transformations

```
let major = [0\31, 5\31, 10\31, 13\31, 18\31, 23\31, 28\31]

fun phrase(steps: list[Interval], dur: Ratio): Music =
    steps.map(s => note(s, dur)).reduce((a, b) => a.seq(b))

fun harmonize(m: Music, interval: Interval): Music =
    m.par(m.transpose(interval))

// build structure
let theme = phrase(major, 1/4)

let section_a = theme.seq(theme.transpose(13\31))
let section_b = theme.invert(0\31).seq(theme.retrograde)
let chorus = theme.harmonize(18\31)

let piece = section_a
    .seq(chorus)
    .seq(section_b)
    .seq(chorus)

// analyze
print("Duration: ${duration(piece)} beats")
print("Pitches used: ${pitches(piece).len}")

// render
piece.realize
    .render(bpm: 108, ref: 262.0hz, sample_rate: 48000)
    .export("structure.wav")
```

## 7.6 Bohlen-Pierce Scale (Non-Octave Tuning)

```
// Bohlen-Pierce: 13 equal divisions of the tritave (3:1)
let bp = edo(13, 3/1)

// A Bohlen-Pierce "triad" (approximating 3:5:7)
let bp_triad = [bp(0), bp(4), bp(6)]

let melody = gen step in cycle([0, 4, 6, 10, 6, 4]) {
    note(bp(step), 1/4)
}

let drone = gen _ in repeat(nil) note(bp(0), 1/1)

let piece = mix([
    voice(melody),
    voice(drone).volume(0.4),
])

piece.realize
    .take_beats(16/1)
    .render(bpm: 72, ref: 262.0hz, sample_rate: 48000)
    .export("bohlen_pierce.wav")
```

## 7.7 Pattern Matching on Music Structure

```
// count all notes in a Music tree
fun count_notes(m: Music): Int = match m {
    Note(_, _, _) => 1,
    Rest(_) => 0,
    Seq(a, b) => count_notes(a) + count_notes(b),
    Par(a, b) => count_notes(a) + count_notes(b),
    Modify(_, inner) => count_notes(inner)
}

// extract all pitches
fun all_pitches(m: Music): list[Interval] = match m {
    Note(p, _, _) => [p],
    Rest(_) => [],
    Seq(a, b) => all_pitches(a).concat(all_pitches(b)),
    Par(a, b) => all_pitches(a).concat(all_pitches(b)),
    Modify(Transpose(i), inner) => all_pitches(inner).map(p => p + i),
    Modify(_, inner) => all_pitches(inner)
}

// compute pitch-class histogram for analysis
fun pc_histogram(m: Music, divisions: Int): list[Int] {
    mut hist = list.fill(divisions, 0)
    for p in all_pitches(m) {
        let pc = cents(p).div(1200.0 / float(divisions)).floor.mod(divisions)
        hist(pc) = hist(pc) + 1
    }
    hist
}

let piece = phrase(major, 1/4).harmonize(18\31)
print("Notes: ${count_notes(piece)}")
print("Pitch classes: ${pc_histogram(piece, 31)}")
```

## 7.8 DFT on Pitch-Class Sets (Complex Number Usage)

```
// Discrete Fourier Transform on a pitch-class set
// Uses complex numbers for spectral analysis of interval content
// (following David Lewin's generalized interval systems)

let pi = 3.14159265358979

fun dft(pc_set: list[Int], n: Int): list[Complex[Float]] {
    gen k in 0..n {
        pc_set.map(p => {
            let angle = -2.0 * pi * float(k * p) / float(n)
            cos(angle) + sin(angle) * 1.0i
        }).reduce((a, b) => a + b)
    }.collect
}

// 12-TET major scale as pitch classes
let major_12 = [0, 2, 4, 5, 7, 9, 11]
let spectrum = dft(major_12, 12)

for k in 0..12 {
    let magnitude = sqrt(spectrum(k).real.pow(2) + spectrum(k).imag.pow(2))
    print("k=${k}: magnitude=${magnitude}")
}
```

## 7.9 A Musical Round (Demonstrating `mix` as Library Function)

```
// "Row, Row, Row Your Boat" as a 4-voice round in just intonation

let scale = [1/1, 9/8, 5/4, 4/3, 3/2, 5/3, 15/8]

fun melody(): Seq[Note] {
    let p1 = [0, 0, 0, 1, 2]          // row row row your boat
    let d1 = [3/8, 3/8, 1/4, 1/8, 3/8]
    let p2 = [2, 3, 4]                 // gently down the stream
    let d2 = [1/4, 1/8, 3/8]
    let p3 = [4, 4, 3, 3, 2, 2, 1, 1, 0]  // merrily...life is but a dream
    let d3 = [1/8, 1/8, 1/8, 1/8, 1/8, 1/8, 1/8, 1/8, 3/8]

    let all_pitches = p1.concat(p2).concat(p3)
    let all_durs = d1.concat(d2).concat(d3)

    gen i in 0..all_pitches.len {
        Note { pitch: scale(all_pitches(i)), duration: all_durs(i), velocity: 80 }
    }
}

// 4-voice round: each voice enters 4 beats after the previous
let stagger = 4/1

let piece = mix([
    voice(melody()),
    voice(melody()).offset(stagger),
    voice(melody()).offset(stagger * 2),
    voice(melody()).offset(stagger * 3),
])

piece.realize
    .render(bpm: 100, ref: 262.0hz, sample_rate: 48000)
    .export("round.wav")
```
