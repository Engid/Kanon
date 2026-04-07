# Kanon

Kanon is a programming language for writing music.

The v0 implementation is in TypeScript for fast iteration.

Kanon is designed to be a language-as-a-library (like Lua) so it's easy to host.

The v0 implementation isn't focusing on being nice-to-host yet, since we are still figuring out what the language _is_.

See the [overview](v0/spec/01-overview.md) for more info.

## Why The Name Fits

Kanon is named after the Pythagorean monochord, the single-string instrument used to demonstrate that musical consonance follows exact numerical ratios such as 2:1, 3:2, and 4:3. That makes it an unusually direct fit for a language centered on exact rationals, interval structure, and composition as mathematics made audible.

The name also carries a few useful secondary meanings:

- It evokes the monochord itself: one simple instrument revealing deep structure.
- It resonates with the musical idea of a canon, which fits Kanon's layering and offset-based composition model.
- It is also used as a Japanese given name with musical associations, which adds a softer musical and human dimension without pulling the project away from its mathematical core.

---

# Spec: 

Currently the specification for the language is in the v0 directory. As elements of the language are more clearly defined and tested, specification documents will begin moving to a top level directory. Changing language features will move through an RFC process with proposals and plans for either modifying the existing spec or adding onto it (sort of like the existing workshop documents).