
## User

all resolved.

I want thought experiment

How far we can push rust type system to prevent this kind hard-to-debug bug from earlier session, i.e. making sure invalid states are irepresentable. Can we?

## Assistant

Most of them. Eleven of the thirteen distinct bugs in this session were type-preventable. But the leverage isn't where people usually reach first (newtypes); it's in deleting API surface so the wrong call has nowhere to live.

## Scoreboard

```
bug                                              types can kill it?
─────────────────────────────────────────────────────────────────────
levels[N] vs levels[N-1]                          yes   (A)
for (lvl, _) binding .0 as if it were an index    yes   (A)
to_delete.extend_from_slice(_output)              yes   (A)
int division truncating, / 0 trapping             yes   (A)
upper/lower bindings swapped (twice)              yes   (B)
upper_level: Some(0) instead of None              yes   (B)
upper_level_sst_ids: vec![]                       yes   (C)
lower_level_sst_ids omitted from the merge        yes   (C)
delete list missing the lower inputs              yes   (D)
extend_from_slice where replace was meant         yes   (D)
output landing in the wrong level                 yes   (D)
─────────────────────────────────────────────────────────────────────
guard duplicating rule 1 -> rule 2 dead code      NO
`>` vs `>=` at the trigger boundary               NO
"L1 is one non-overlapping sorted run"            NO
```

## A. Tag the scalars

A newtype is a struct with exactly one field. Mechanically it is *nothing*:

```rust
struct Level(usize);

// Level(3)  in a register:   0x0000000000000003
// 3usize    in a register:   0x0000000000000003
//           identical bits, identical instruction, zero allocation.
```

The wrapper lives only in the compiler's type table and is erased before a single machine instruction is emitted. What you buy is that `Vec<T>` does not implement `Index<Level>`, so this stops compiling:

```rust
for (lvl, ids) in snapshot.levels.iter() { ... }   // lvl: &Level now
snapshot.levels[*lvl]                              // ERROR: Vec cannot be indexed by Level
```

The `- 1` doesn't vanish. It moves to exactly one place in the entire program:

```rust
impl LsmStorageState {
    fn files(&self, l: Level) -> &Vec<usize> {
        &self.levels[l.0 - 1].1        // the only -1 that exists anywhere
    }
}
```

You wrote that `- 1` wrong in four different places over this session. One place is a place you can stare at until it's right.

Go bridge, with the divergence: Go has `type Level int`, same idea. But Go leaks — `var l Level = 3` compiles (untyped constant conversion) and `levels[l]` compiles too, because slice indexing accepts any integer type. Rust's version holds: you must write `Level(3)` explicitly, and indexing is a hard error. Go's newtype is a suggestion; Rust's is a wall.

The same trick would have killed your last bug outright:

```rust
fn apply_compaction_result(..., output: &[NewSst]) -> (LsmStorageState, Vec<DeadSst>)

to_delete.extend_from_slice(output)
//                          ^^^^^^ expected &[DeadSst], found &[NewSst]
```

Two `usize` lists with opposite meanings, indistinguishable to the compiler as written. Give the roles names and the confusion becomes a compile error.

And the ratio:

```rust
struct FileCount(usize);
impl FileCount {
    fn ratio_over(self, denom: FileCount) -> f64 {
        self.0 as f64 / denom.0 as f64   // truncation and the /0 trap both gone,
    }                                    // in one place, forever
}
```

## B. Make the bad pairing unconstructible

`upper` and `lower` are not two independent numbers. They're an adjacent pair, and one determines the other. So stop letting anyone supply both:

```rust
enum Upper { L0, Level(Level) }          // Some(0) is now unrepresentable

struct Pair { upper: Upper, lower: Level }   // fields private

impl Pair {
    fn below(upper: Upper) -> Pair {         // the ONLY constructor
        let lower = match upper {
            Upper::L0        => Level(1),
            Upper::Level(l)  => Level(l.0 + 1),
        };
        Pair { upper, lower }
    }
}
```

You swapped these bindings twice. With `Pair::below()` as the only door in, "swapped" is not a state that exists. The `+ 1` is written once, by the person who is thinking about direction at that moment, rather than by the person writing a loop body at 2am.

Note what `enum Upper` buys separately: `Option<usize>` let you write `Some(0)`, a value the whole codebase agreed was meaningless but the type permitted. `Upper::L0` is the same information with the nonsense removed.

## C. Fields nobody can write

In Rust, a struct with private fields **cannot be built with literal syntax from another module**. Not "shouldn't" — the `Task { .. }` expression is a hard error. So:

```rust
pub struct Task { upper: Upper, lower: Level, upper_ids: Vec<usize>, lower_ids: Vec<usize> }

impl Task {
    pub fn new(snap: &LsmStorageState, upper: Upper) -> Task {
        let Pair { upper, lower } = Pair::below(upper);
        Task {
            upper_ids: snap.files_of(upper).clone(),   // never empty by accident
            lower_ids: snap.files(lower).clone(),      // never omitted
            upper, lower,
        }
    }
}
```

`upper_level_sst_ids: vec![]` survived four rounds of this session because a struct literal happily accepts a plausible-looking wrong value for every field. Once the ids are *derived from the snapshot* rather than *supplied by the caller*, both bugs are gone: there is no expression a caller can write that produces a task with the wrong files in it.

Go does this identically with lowercase fields plus a `NewTask` constructor. This is the one pattern that transfers 1:1.

## D. Delete the choice

This is the biggest win and it isn't a type at all. Look at the signature you spent this whole session fighting:

```rust
fn apply_compaction_result(&self, snap, task, output) -> (LsmStorageState, Vec<usize>)
```

That `Vec<usize>` asks the implementor a question whose answer is **already fully determined by `task`**. Dead files = every input to the merge = `task.upper_ids ++ task.lower_ids`. Always. There is no policy in it, no per-controller variation. The parameter exists only as an opportunity to get it wrong, and you got it wrong three separate ways: `output`, upper-only, and upper-plus-output.

```rust
fn apply_compaction_result(&self, snap, task, output) -> LsmStorageState
```

and the caller in `compact.rs` computes the delete list itself from the task it already holds. One bug class, deleted by removing a return value.

Same move for `extend` vs `replace`. Make `levels` private and expose only the operation that's legal:

```rust
impl LsmStorageState {
    fn replace_level(&mut self, l: Level, files: Vec<NewSst>) { ... }
}
```

There is no `.extend()` reachable from outside, so "appended where I meant to overwrite" stops being an expressible mistake. And `replace_level(Level, ...)` means the output physically cannot land in L2 when the task said L1 — the level came out of `Pair`, not out of arithmetic at the call site.

## The wall

**Const generics won't save you.** The tempting move is `Level<const N: usize>` with `fn compact<const N: usize>(u: Level<N>, l: Level<{N+1}>)`, making adjacency a compile-time fact. Two reasons it dies: `{N+1}` in type position needs `generic_const_exprs`, unstable and likely to stay that way for years; and `max_levels` is a *runtime config value*, so the compiler doesn't know how many levels exist. Don't chase this.

**The sorted-run invariant is genuinely out of reach.** "Every file in L1 has a key range disjoint from and ordered against its neighbours" is a property of bytes inside files on disk, discoverable only by reading them. Encoding it would need a type that depends on a runtime value — dependent or refinement types, which Rust does not have and is not getting. Liquid Haskell, F*, and Idris can express it; Rust cannot. Your tools here are `debug_assert!` at the boundary of `replace_level` and property tests that run compactions against a reference model.

**Pure logic is pure logic.** `>` vs `>=`, and a guard whose condition happens to be the complement of the next `if`, are semantic errors in code that is perfectly well-typed. No type discipline sees them. Tests do — and note that the compaction simulator *did* catch both, immediately, which is why it exists.

## Why the real code doesn't do any of this

`levels: Vec<(usize, Vec<usize>)>` is untyped because one `LsmStorageState` serves both leveled mode (where `.0` is a level number) and tiered mode (where `.0` is an SST id and positions shift). Typing it properly means an enum over two layouts, which means every consumer matches on it, which means a tutorial codebase becomes twice as long before the reader has written a line. That's a deliberate trade for a teaching repo, and it's the trade that generated most of this session's debugging.

The compressed rule: **a function taking two arguments of the same type is a coin flip the compiler can't referee.** `f(usize, usize)` is swappable. `f(Pair)` is not. Every fix above is a variation on collapsing several loose scalars into one value that knows what it means.

