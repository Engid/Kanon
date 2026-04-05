# 2. Lexical Grammar

## 2.1 Comments

```
// single-line comment

/* multi-line
   comment */
```

## 2.2 Identifiers

Identifiers start with a letter or underscore, followed by letters, digits, or underscores. Type names are capitalized by convention. Variable and function names are lowercase or snake_case.

```
my_var   _temp   Note   Seq   EDO19   x2
```

## 2.3 Keywords

```
fun     let     mut     type    enum
match   if      else    for     in
gen     yield   return  true    false
nil     import  as      where
and     or      not
```

## 2.4 Numeric Literals

### Integers

```
42          // decimal
0xFF        // hexadecimal
0b1010      // binary
0o77        // octal
1_000_000   // underscores for readability
```

### Rationals (exact)

The `/` operator between two integer literals produces an exact rational value. This is a literal form, not a division operation.

```
3/2         // Ratio: exactly three-halves
5/4         // Ratio: the just major third
7/1         // Ratio: integer promoted to rational
22/7        // Ratio: exact (not a float approximation of pi)
```

### Floats (inexact)

Any numeric literal with a decimal point is a float. Floats use IEEE 754 double precision.

```
3.14        // Float
1.0         // Float (not integer, not rational)
6.02e23     // Float with exponent
```

### Complex

Postfix `i` on any numeric literal produces the imaginary component. Complex numbers have either rational or float components depending on the literal type.

```
2i            // Complex: 0 + 2i
3/2 + 1/2i    // Complex[Ratio]: (3/2) + (1/2)i
1.5 + 0.5i    // Complex[Float]: 1.5 + 0.5i
```

### EDO Step Literals

The `\` operator between two integer literals produces an EDO (Equal Division of the Octave) step. This uses the established xenharmonic community convention where `n\d` means "step n of d-EDO."

```
7\12        // step 7 of 12-EDO (≈ perfect fifth, 700 cents)
1\12        // step 1 of 12-EDO (semitone, 100 cents)
6\19        // step 6 of 19-EDO
9\31        // step 9 of 31-EDO (neutral third)
0\53        // unison in 53-EDO
```

The type of an EDO literal is `Edo`, carrying its division count at runtime.

### Unit-Typed Literals

Postfix `hz` and `c` on float literals produce frequency and cents values respectively.

```
440.0hz     // Frequency in Hertz
261.63hz    // Middle C
100.0c      // Interval of 100 cents (one 12-EDO semitone)
701.955c    // A just perfect fifth in cents
```

## 2.5 String Literals

Double-quoted strings with interpolation via `${}`.

```
"hello"
"pitch: ${n.pitch}"
"line 1\nline 2"
"the interval is ${cents(3/2)}c"
```

## 2.6 Operators

```
// arithmetic
+   -   *   /   %

// comparison
==  !=  <   <=  >   >=

// logical
and   or   not

// assignment
=

// pipe (available, UFCS is preferred for chaining)
|>

// range
..      // exclusive end: 0..10 is 0,1,2,...,9
..=     // inclusive end: 0..=10 is 0,1,2,...,10
```

### Operator Notes

- `/` between two integer literals is a **rational literal**, not division. `3/2` is the ratio three-halves. For float division, at least one operand must be a float: `3.0 / 2.0`. For truncating integer division, use the `div()` function.
- `\` between two integer literals is an **EDO step literal**. It is an error if either operand is not an integer literal.
- `i` as a postfix on a numeric literal produces an **imaginary component**.
- `hz` and `c` as postfix on float literals produce **Hz** and **Cents** values.
