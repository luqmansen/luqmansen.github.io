

## Assistant

Fair. I gave you engineering discipline, not the type system. Your instinct is right, and it happens to be the one exotic pattern in Rust that's genuinely load-bearing rather than a party trick. Let me build it, then show you the exact line where it dies, then correct the TypeScript premise, because that's where the reach is off.

## Your idea, built

Right now the disjointness invariant lives in a doc comment (`concat_iterator.rs:29`):

```rust
/// Concat multiple iterators ordered in key order and their key ranges do not overlap.
pub struct SstConcatIterator { ... }

impl SstConcatIterator {
    pub fn create_and_seek_to_first(sstables: Vec<Arc<SsTable>>) -> Result<Self>
    //                                        ^^^^^^^^^^^^^^^^^ any Vec at all
}
```

A raw `Vec` plus a sentence in English. Move the sentence into the type:

```rust
mod sealed { pub trait Sealed {} }          // nobody outside can add layouts

pub trait Layout: sealed::Sealed {}
pub struct Overlapping;                     // L0: files written at different times
pub struct Disjoint;                        // L1+: one sorted run

impl sealed::Sealed for Overlapping {}
impl sealed::Sealed for Disjoint {}
impl Layout for Overlapping {}
impl Layout for Disjoint {}

pub struct Run<L: Layout> {
    files: Vec<Arc<SsTable>>,
    _l: PhantomData<L>,
}
```

Mechanically, `PhantomData<L>` occupies zero bytes:

```
Run<Disjoint>       in memory        Run<Overlapping>    in memory
┌──────────────┐                     ┌──────────────┐
│ ptr   8 bytes│                     │ ptr   8 bytes│
│ len   8 bytes│                     │ len   8 bytes│    byte-identical.
│ cap   8 bytes│                     │ cap   8 bytes│    24 bytes each.
│ _l    0 bytes│  <- nothing here    │ _l    0 bytes│
└──────────────┘                     └──────────────┘
     24 bytes                             24 bytes
```

`Overlapping` and `Disjoint` are unit structs: no fields, size 0, never instantiated, never appear in the binary as data. They exist purely as a tag in the compiler's type table, gone before codegen. Now:

```rust
impl Run<Disjoint> {
    fn scan(&self) -> SstConcatIterator { ... }   // binary search, one file open at a time
}
impl Run<Overlapping> {
    fn scan(&self) -> MergeIterator<SsTableIterator> { ... }   // heap over every file
}
```

`SstConcatIterator` is now unreachable from L0. The `extend_from_slice` bug you shipped into L1 — appending files whose ranges overlap what's already there — becomes a compile error if the only mutator is:

```rust
impl Run<Disjoint> {
    fn replace(&mut self, proven: Run<Disjoint>) { self.files = proven.files }
    // no extend. no push. no &mut Vec escape hatch.
}
```

## Typestate: the type as a receipt

Push it one step. How do you ever *get* a `Run<Disjoint>`? Exactly one door:

```rust
impl Run<Disjoint> {
    fn from_merge(inputs: MergeIterator<SsTableIterator>) -> Run<Disjoint> {
        // the merge itself is what establishes ordering. The output of a
        // k-way merge into sequential SSTs is disjoint by construction.
    }
}
```

The type is not a claim; it's a receipt that the merge ran. This is the real content of "make invalid states unrepresentable" in Rust — not newtypes, but **a private field plus a constructor that is the only proof-producing step**. Every serious Rust database does this. `Run<Disjoint>` is unforgeable outside the module.

## The genuinely exotic tier

You want adjacency — "you may only compact into the level directly below" — as a compile-time fact. That's type-level naturals, Peano-encoded, and it works on **stable Rust today** (this is what the `typenum` crate is):

```rust
struct Z;                                   // zero
struct S<N>(PhantomData<N>);                // successor

trait Nat { const N: usize; }
impl Nat for Z            { const N: usize = 0; }
impl<P: Nat> Nat for S<P> { const N: usize = P::N + 1; }

type L1 = S<Z>;
type L2 = S<L1>;
type L3 = S<L2>;

struct Level<N: Nat>(Run<Disjoint>, PhantomData<N>);

fn compact<N: Nat>(upper: &Level<N>, lower: &Level<S<N>>) -> Level<S<N>>
//                              ^^                  ^^^^
//                              adjacency is in the signature
```

```rust
compact(&l1, &l2)   // compiles
compact(&l1, &l3)   // ERROR: expected Level<S<S<Z>>>, found Level<S<S<S<Z>>>>
compact(&l2, &l1)   // ERROR: same, in the other direction
```

Both bugs you hit — swapped upper/lower, and the off-by-one destination — are now rejected by the compiler with no assertion, no test, no runtime cost. `Level<L2>` and `Level<L3>` have identical layout; the whole apparatus evaporates at codegen.

**Now the line where it dies.** Your rule 2:

```rust
for i in 1..self.options.max_levels {
    //        ^^^^^^^^^^^^^^^^^^^^^ a runtime value read from a config struct
    check_ratio(level(i), level(i + 1));
}
```

A loop body has exactly one type. Pass 1 needs `Level<L1>, Level<L2>`; pass 2 needs `Level<L2>, Level<L3>`. Those are different types, and there is no function `fn level(i: usize) -> Level<?>` because Rust cannot select a type from a runtime integer. Monomorphization happens before `main` runs; `max_levels` isn't known until after.

Two escapes, both real, both expensive:

```
(a) macro that unrolls the loop for a fixed maximum
(b) heterogeneous list of levels, walked by trait recursion (frunk-style HList)
```

Either way `max_levels` must become a compile-time constant. `SimpleLeveledCompactionOptions.max_levels` stops being a config field, and the compaction simulator's `--max-levels 3` flag stops existing. **That is the price, and it is not a taste question.** The type system's boundary is the compile-time/runtime line, and this invariant sits on the wrong side of it.

## The TypeScript premise

"Rust and TypeScript's types aren't too far off" is where the reach is off, and the difference is mechanical, not a matter of degree.

```
TypeScript                              Rust
────────────────────────────────────────────────────────────────────────
types are ERASED.                       generics are MONOMORPHIZED.
  tsc computes them and emits              every distinct instantiation
  nothing. A 50,000-step type-level        stamps out a real copy of the
  program costs compile time and           code. Type-level computation
  zero bytes of output.                    that touches values emits symbols.

conditional types:                      no conditional types.
  T extends U ? X : Y                      no "if T: Trait then X else Y"
  arbitrary branching on any type          on stable. Specialization is
                                           unstable and has been for a decade.

negative reasoning is free:             negative reasoning is FORBIDDEN.
  Exclude<T, U>, T extends never           impl<T> Foo for T where T: !Bar
                                           does not exist and will not.

literal types are computable:           const generics can't do arithmetic.
  "abc", 42, template literals,            {N + 1} in type position needs
  tuple-length arithmetic                  generic_const_exprs, unstable.
```

Capability isn't the gap — Rust's trait resolver is Turing-complete too; people have written type-level Brainfuck interpreters with it. The gaps are deliberate design, and the negative-reasoning one is worth understanding because it's the biggest:

Rust guarantees that a downstream crate adding `impl Foo for Bar` can never break an upstream crate that already compiled. If you could write "apply this impl only when `T` does *not* implement `Foo`", then anyone anywhere in your dependency graph adding an impl could silently flip which branch a distant crate resolved to. Coherence forbids it. TypeScript has no separate-compilation contract to honour — every build re-checks everything from source — so `Exclude<T, U>` is free for it and structurally impossible for Rust.

That, plus monomorphization, is why Tetris-in-types exists in TS and doesn't in Rust. In TS the type-level program is a compiler-time computation producing nothing. In Rust it wants to produce machine code.

## Verdict

The `Run<Disjoint>` / `Run<Overlapping>` split is worth shipping. It's cheap, it's zero-cost at runtime, it converts `concat_iterator.rs:29`'s doc comment into a checked fact, and it kills the class of bug where L1 quietly stops being a sorted run — which is the one bug in this session that would have produced *silently wrong reads* rather than a panic.

The Peano-level tier is a genuine, working technique that is wrong here for one concrete reason: `max_levels` is runtime configuration. Change that and it becomes viable.

And the invariant you'd most want — "these files' key ranges are actually disjoint" — remains out of reach for both Rust and TypeScript, because it's a property of bytes on disk. `Run<Disjoint>` doesn't prove disjointness; it proves *a merge produced this*, and you trust the merge. That gap is where refinement types (Liquid Haskell, F*) live, and neither of these languages goes there.

## User

To summarize, basically what you're doing is making all thinkable unit of concepts to be its own type with its own method, so there's no need to infer things on the fly, am I right?

## Assistant

Mostly, but the summary is missing the half that does the work, and it's the half that separates this from documentation-with-better-syntax.

"One type per concept" is the visible part. The load-bearing part is **one door per type**.

```
type as a label (worthless)            type as a receipt (the real thing)

anyone, anywhere, writes:              module boundary
                                       ┌────────────────────────────────┐
  Run { files: whatever,               │  files: private                │
        _l: PhantomData }              │                                │
        ^^^^^^^^^^^^^^^^               │  fn from_merge(..)             │ ◀─ the only
        asserts Disjoint,              │      -> Run<Disjoint>          │    way in
        proves nothing                 └────────────────────────────────┘
                                         outside this module, the literal
                                         `Run { .. }` is a compile error
```

`struct Disjoint;` on its own is a sticker. What makes `Run<Disjoint>` mean something is that the fields are private and exactly one function produces the value, and that function is the merge itself. The type isn't the proof; it's the label on a proof that already happened. Any design where a caller can conjure the type directly has bought nothing.

Second refinement: it's less "each type gets its own methods" and more **the wrong operation is absent**.

```
Run<Disjoint>:   .replace(Run<Disjoint>)    exists
                 .extend(Vec<SsTable>)      does not exist, anywhere, for anyone
```

Your `extend_from_slice` bug wasn't caught by a method being present. It was caught by there being no such method to call. Absence is the instrument.

Your last clause is the right instinct though, sharpened: you're not helping the compiler understand your program. The compiler understands nothing extra. You're **relocating a fact from a place it can't read to the one place it can** — out of a doc comment, out of a reviewer's head, into the type table. Same fact, checkable location.

And the cost, so this isn't a blanket rule: every concept-as-type adds a conversion at every boundary it crosses. Spend it where a violation is *silent*. Disjointness qualifies — break it and reads return stale values with no crash. "This `usize` is a file count" mostly doesn't; get that wrong and a test fails loudly on the first run.
