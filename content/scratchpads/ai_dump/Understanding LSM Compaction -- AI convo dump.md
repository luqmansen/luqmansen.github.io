
I thought the idea of compaction is to merge bunch of SSTs into fewer, bigger SSTs.


But seems it's not just the case. There's also a case where num of sst in == sst out, but just there's no overlapping key ranges.


Why don't we just have fewer bigger SSTs?

## Assistant

Your premise has one word too many. Compaction's job is not "fewer files." It's **"remove overlap, and keep each run bounded."** Fewer files is sometimes a side effect, not the goal.

Two separate wins are hiding here, and they pull in opposite directions on "how big should one SST be."

## Win 1: kill overlap (the read win)

L0 SSTs are just memtable dumps. Each one spans the whole keyspace, so they **overlap each other**:

```
L0:  [a .............. z]     <- SST 3
     [a .............. z]     <- SST 2
     [a .............. z]     <- SST 1
```

A point lookup for key `m` must probe **every** L0 SST, because any of them could hold `m`. Three SSTs = three bloom checks + three binary searches.

After compaction into L1, the runs are **disjoint**:

```
L1:  [a-f] [g-l] [m-r] [s-z]     <- non-overlapping, sorted
                  ^^^^^
```

Now a lookup for `m` binary-searches the SST boundaries, picks the **one** SST whose range covers `m`, and probes only that. This is the "same count in, same count out, but no overlap" pass you saw: the count barely changed, but the *overlap* went from total to zero. That is the whole point of that pass.

## Win 2: keep runs bounded (the write-amp win) — this is your real answer

Now, why not collapse L1 into one giant `[a-z]` file? Because of what the **next** compaction costs.

```
ONE GIANT SST:
  L0 flush arrives: [d,f]   (2 MB)
  L1:               [ a ........................ z ]   (10 GB, one file)

  merge [d,f] in  ->  read 10 GB, rewrite 10 GB
                      to absorb 2 MB of new data.        <-- catastrophic write amp


BOUNDED, DISJOINT SSTs:
  L0 flush arrives: [d,f]   (2 MB)
  L1:  [a-c] [d-f] [g-i] [j-l] ... [x-z]   (each ~target_sst_size, disjoint)
             ^^^^^
  merge [d,f] in  ->  only [d-f] overlaps the incoming range.
                      read ~2 MB + rewrite ~2 MB.
                      every other SST: NOT touched. Just keep its
                      pointer in the manifest unchanged.
```

Because the runs are disjoint and size-capped, an incremental merge only rewrites the **slice** whose key range collides with the new data. Cost drops from `O(entire level)` to `O(overlapping SSTs)`. One giant file maximizes write amplification: any tiny update forces a full rewrite.

## So the size cap is deliberate

The `target_sst_size` cap is what *creates* the "many bounded SSTs" layout. During compaction the builder streams keys and flushes a fresh SST every time it hits the cap, which is also why a single compaction job stays bounded in memory instead of buffering 10 GB.

Net: "fewer bigger SSTs" buys you nothing on reads (one disjoint SST is already touched per level regardless of size) and wrecks write amplification. So you keep them split. The pass that confused you is paying a small cost now (make ranges disjoint, hold the size cap) so every future merge stays local.

## User

Then, i don't understand the point of leveled/tiered compaction, which always produce bigger sorted file.

Why? Also fundamentally i don't understand the diff betweeen these 2

## Assistant

Drop the word "bigger." That's not what separates them. Both build sorted runs. What separates them is **how many sorted runs you tolerate at each level**, and that one knob sets the entire cost profile.

First, one primitive both share.

## The unit: a "sorted run"

A sorted run = data whose keys are globally sorted and non-overlapping. It can be one SST, or many SSTs split by the size cap with disjoint ranges. Either way, a lookup touches exactly **one** SST inside a run (binary search the boundaries).

So the only real question a compaction strategy answers is: **how many sorted runs sit at each level, and what triggers a merge?**

## Tiered: many runs per level, merge whole runs

Each memtable flush becomes a new run. Runs pile up. Every run spans roughly the whole keyspace, so they **overlap each other**:

```
tier 0 (newest):  [ a ............ z ]   run
tier 1:           [ a ............ z ]   run
tier 2:           [ a ............ z ]   run
tier 3 (oldest):  [ a ............ z ]   run
```

Trigger: "too many runs" (or a size ratio). Action: pick several whole runs and merge them into one:

```
tier 0,1,2,3   ── merge all ──▶   [ a ............ z ]   one bigger run
```

A read for key `m` must probe **one SST in every tier**, because any tier could hold the newest `m`. Old versions of `m` survive in older tiers until a big merge finally drops them.

## Leveled: exactly one run per level, merge a slice down

Each level L1..Ln is a **single** sorted run (disjoint SSTs). Levels grow by a fixed multiplier (say 10x):

```
L0:  [a..z][a..z][a..z]   overlapping flushes (the one exception)
L1:  [ a ........ z ]     ONE run,  ~10 MB target
L2:  [ a ........ z ]     ONE run,  ~100 MB target
L3:  [ a ........ z ]     ONE run,  ~1 GB target
```

Trigger: a level exceeds its size target. Action: take some SSTs from Li and merge them into **only the overlapping slice** of Li+1:

```
L1:  ... [g-i] ...              pick [g-i]
L2:  [a-f][g-i][j-l]...   ──▶   rewrite just L2's [g-i] slice, rest untouched
```

A read for `m` touches one SST per level. Runs never overlap within a level, so no version ambiguity except across levels (newer level wins).

## The fundamental difference in one line

**Tiered = many overlapping runs per level, merged whole, rarely. Leveled = one disjoint run per level, merged a slice at a time, continuously.**

## Why both exist: the amplification triangle

You cannot minimize all three costs at once. Pick two:

```
                 write amp        read amp         space amp
                 (rewrites of     (SSTs probed     (dead/dup
                  the same key)    per read)        copies kept)
  tiered           LOW              HIGH             HIGH
  leveled          HIGH             LOW              LOW
```

**Why leveled's write amp is high:** to push a 10 MB slice from L1 into L2 (which is 10x denser), the overlapping L2 portion is ~100 MB. Writing 10 MB of new data forces rewriting ~100 MB. Repeat at every level: each key gets rewritten roughly `multiplier` times per level on its way down. Clean reads, expensive writes.

**Why tiered's write amp is low:** you only merge whole runs occasionally, so a key is rewritten about once per level total, not `multiplier` times. But you pay for it in reads (probe every tier) and space (many overlapping runs and stale versions coexist until the next big merge).

So the "point" was never "make bigger files." It's: raw flushes are many small overlapping runs, which make reads slow and never reclaim space from overwritten/deleted keys. Merging fixes that. **Tiered fixes it lazily** (write-optimized, for ingest-heavy workloads). **Leveled fixes it eagerly** (read/space-optimized, for query-heavy workloads). Same enemy, opposite ends of the triangle.

## User

i don't understand the point of tiered runs. Isn't basically just the same as L0 SSTs files if i have to proble all of the file for every read?

Also for leveled compaction, isn't that i need to probe one file for every level anyway? because it could be anywhere literally

So what's the difference?

## Assistant

Two separate confusions. Let me take them in order, because the answer to the second one is going to surprise you: **you're basically right about reads.**

## Q1: tiered vs a raw L0 pile — the difference is COUNT GROWTH

A tier is not a flush. A tier is a **merged run**, and merging keeps the *number* of runs bounded. That is the entire point.

```
raw L0 pile, no compaction:
  ingest 10 GB, 2 MB per flush  ->  5000 overlapping 2 MB files
  read key m  ->  probe 5000 files          # count grows with data, forever

tiered:
  ingest 10 GB  ->  ~6 runs:  [2MB][8MB][32MB][128MB][2GB][8GB]
  read key m  ->  probe 6 runs              # count stays ~constant
```

Both "probe one per run." But the raw L0 pile's run count is `data / flush_size` and grows without bound. Tiered's run count is capped by the merge trigger (`num_tiers`), so it stays ~constant no matter how much you ingest. That 8 GB tier is **one sorted run** (disjoint SSTs inside, one probe). If you'd left it as raw L0 it would be 4000 overlapping files. Same bytes, 4000 probes vs 1.

So tiered isn't "L0 but pointless." It's "L0 with the file count held down by periodic merging."

## Q2: leveled — you're right, it IS ~one probe per level

Yes. A key "could be anywhere," so you walk levels top-down and check one SST at each. On raw probe count, leveled and tiered are in the same ballpark (both ~log-many runs). **Reads are not where these two strategies differ.** Trust that instinct.

Two things soften "probe every level," and they apply to both strategies:

**Bloom filters.** At each level you check the bloom first, in memory, nanoseconds. It answers "definitely not here" for the levels that don't hold the key, so no disk read there. "Could be anywhere" collapses to "the one level whose bloom says *maybe*" plus a handful of cheap in-memory rejections. Effective disk reads for a point lookup ≈ 1, not `#levels`.

Bloom filters do NOT help range scans (you can't bloom-check a range). A scan must merge across every run. Here leveled wins because it has *fewer* runs to merge (one per level) than tiered (runs can stack up).

## So what actually differs? Write amp and space amp.

Since reads are roughly a wash, the real reason to pick one is the other two costs:

```
                write amp              space amp
                (times a key is        (extra copies /
                 rewritten)             dead versions on disk)
  tiered        LOW                    HIGH
  leveled       HIGH                   LOW
```

**Space amp, mechanically:** tiered lets overlapping runs coexist. The old value of `m` sits in an old tier while the new value sits in a new tier — both on disk until a merge spanning both finally drops the old one. Right before a big merge you can be holding ~2x the live data. Leveled makes each level 10x the one above, so the bottom level dominates and each key mostly exists once; space overhead ≈ 1.1x. Pick leveled when disk cost matters.

**Write amp, mechanically:** leveled pushes a slice from Li into Li+1, which is 10x denser, so writing 10 MB of new data rewrites ~100 MB of overlap — each byte rewritten ~`multiplier` times per level. Tiered merges whole runs of *similar* size only occasionally, so each byte is rewritten ~once per level. Pick tiered when write/ingest throughput matters.

Same run-bounding machinery, opposite ends of the write-vs-space tradeoff. Reads were never the deciding factor.

## User

doubts and questions

> Reads are not where these two strategies differ
your prev graph
```
write amp        read amp         space amp
(rewrites of     (SSTs probed     (dead/dup
 the same key)    per read)        copies kept)
tiered           LOW              HIGH             HIGH
leveled          HIGH             LOW              LOW

```

on previous graph, you literally shows 

> Leveled makes each level 10x the one above, so the bottom level dominates and each key mostly exists once

What? Isn't key can be anywhere within any level? Suppose that it's a update intensive write, many keys will naturally span over many levels (eg 6 levels). What exactly exists mostly once????

> Tiered merges whole runs of *similar* size only occasionally, so each byte is rewritten ~once per level. Pick tiered when write/ingest throughput matters.

I don't quite get it. how come it's rewritten once per level? Isn't big merge effectoively rewrite all bytes all over again? Can you give me illustration/sample?

## Assistant

You caught a real sloppiness in objection 1. Let me fix that first, then the two mechanical ones.

## 1. I overstated it. Reads *do* differ; the table was right.

My "reads are not where these differ" was wrong as written. Correct version: tiered read amp **is** higher, because it holds **more runs**.

```
leveled:  1 run per level     ->  #runs = #levels        ≈ 5
tiered:   several runs stack   ->  #runs = num_tiers      ≈ 10
          before merging
```

A read walks every run, so tiered walks ~2x as many. That part of the table stands.

What I *should* have said: for a **point lookup**, the bloom filter at each run answers "definitely not here" in memory, so both strategies do ~1 actual disk read regardless of run count. The read-amp gap is real but mostly costs cheap in-memory bloom checks, and it bites hardest on **range scans** (no bloom possible, must merge all runs). So reads differ, just less *decisively* than write/space amp. That was the point I mangled. Sorry.

## 2. "Exists mostly once" — I meant the BULK, not that dups can't happen

You're right: an updated key has stale versions scattered across levels. That's not what "mostly once" was about. Two different things:

```
6-level leveled tree, T=10.  Level sizes:
  L1 = 1 MB      \
  L2 = 10 MB      |  upper levels: ~10% of total bytes
  L3 = 100 MB    /
  L4 = 1 GB      \
  L5 = 10 GB      |  bottom-heavy: L6 alone ≈ 90% of all bytes
  L6 = 100 GB    /
```

"Bulk of distinct keys lives deduplicated in L6" and "hot key `x` has stale copies in a few levels" are both true, because the stale copies are trapped in the **tiny** upper levels:

```
update-heavy: key x written 6 times, newest first
  memtable / L0:  x=v6
  L1:             x=v5        <- these upper levels are TINY (MBs)
  L2:             x=v4
  ...
  L6:             x=v1        <- the settled one, in the 100 GB bottom

  when L1's slice compacts DOWN into L2, and both hold x,
  the merge keeps v5, DROPS v4.  duplicates collapse as data descends.
```

So the stale versions (v4, v5) sit in levels that are 1/1000 of the tree and get dropped the moment their slice is compacted down. Leveled is **eager** (continuous downward merging), so a key never accumulates 6 live copies. Sum the geometry: total on disk = `1 + 1/10 + 1/100 + ...` ≈ **1.11×** the live data. That is what "mostly once" meant: the extra copies are bounded to ~11%, confined to the small top.

Tiered has no such confinement. Its runs are all comparable in size, and an old version sits in an 8 GB run until a big merge spanning it fires. So the stale copies are a large fraction, not a tiny top slice. Space amp ~2x.

## 3. Tiered write amp — "once per level" is AMORTIZED over a byte's life

Your objection is correct *about one merge*: a big merge does rewrite everything in it. The trick is that a byte **participates in only one merge per size class**, then sits untouched until it climbs to the next class. Trace one byte `b` (T=4: merge 4 runs of size S into one run of size 4S):

```
    runs at each size          b's fate
    ------------------         --------
    [2MB] ← b enters
    [2MB]
    [2MB]
    [2MB]   ─ merge 4×2MB ─▶ [8MB]     b copied  (rewrite #1)

    [8MB] ← b sits here, untouched, while 3 OTHER 8MB runs form
    [8MB]
    [8MB]
    [8MB]   ─ merge 4×8MB ─▶ [32MB]    b copied  (rewrite #2)

    [32MB]  ...                        b copied  (rewrite #3) ...
```

`b` is copied once per size class. Number of size classes to hold N bytes = `log_T(N)` ≈ number of levels. So over its whole life, `b` is rewritten ≈ **once per level**. Each merge is big, but `b` is in only one merge per class.

Now compare the two merge *shapes* side by side, one step each:

```
TIERED merge:  many EQUAL-size runs -> one run
  read  4 × 8MB = 32MB
  write 32MB
  every byte written exactly once this step        write amp ≈ 1×

LEVELED merge:  small source + dense destination
  L1: [==10MB==]                 (source)
  L2: [============100MB============]   (destination, T× denser)
  read 10MB + 100MB, write 110MB
  wrote ~100MB just to absorb 10MB of new data     write amp ≈ 10× (= T)
```

That is the whole write-amp gap. Tiered merges runs of the **same** size, so a byte is copied once and moves up a class. Leveled merges a small slice into a **T-times-bigger** destination, so absorbing X bytes forces rewriting ~T·X bytes of the level below, at every level. Multiply out:

```
  tiered  write amp ≈ #levels          ≈ log_T(N)
  leveled write amp ≈ T × #levels      ≈ T · log_T(N)   → ~T times more
```

So "big merge rewrites all bytes" is true, but in tiered each byte is only *in* such a merge `#levels` times total, and each of those merges writes it exactly once. Leveled writes far more because every downward step re-writes the dense level below to absorb a thin slice.

## User

Now can you remind me again why lower the level, the bigger the size of the file is?

i thought on leveled compaction does not need to rewrite the whole file, but just slice out small subset of files that overlap with the range about to get merged 

you said earlier that bigger files has no point and only makes things hard?

## Assistant

You're merging two different meanings of the word "big." Separating them dissolves the whole contradiction.

## Two kinds of "big"

```
"big FILE"          = the size of one SST        -> CAPPED (target_sst_size),
                                                    same at every level
"big LEVEL"         = total bytes in the level   -> GROWS by T going down
```

A level is **not** one giant file. It is a row of equal, bounded SSTs. Lower levels have **more boxes**, not bigger boxes:

```
each box = one SST, all ~same capped size (target_sst_size)

L1 (total ~256 MB):  [a-c][d-f][g-i][j-l]                    4 boxes
L2 (total ~2.5 GB):  [a-b][b-c][c-d]......................   ~40 boxes
L3 (total ~25 GB):   [aa-ab][ab-ac]........................  ~400 boxes
```

So "lower level is bigger" = "holds more SSTs / more total bytes," never "is one enormous file." The turn-1 warning ("one giant SST is bad") was about collapsing a level into a **single** file. Leveled never does that. The per-file cap holds everywhere. No contradiction with slice-rewrite: you still touch only the overlapping boxes.

## Reconciling "only a slice" with "write amp ≈ T"

Both are true at once, and here's the mechanism. You push one SST down from L1. It covers, say, range `[d-f]`. L2 is 10x denser, so over that **same key range** L2 packs ~10 SSTs:

```
push one L1 SST, range [d-f]:

L1:  ............[d-f]............                1 box  (~256 MB over d..f)

L2:  ...[d.][d.][.e][.e][e.][ef][f.]...          ~10 boxes over d..f
        └────────── overlap ──────────┘          (~2.5 GB, because L2 is 10× denser)

merge:  read [d-f]  +  those ~10 L2 boxes
        write ~10 boxes back
        touched 10 boxes,  NOT all 400 boxes of L2   ← slice, not whole level
        wrote ~10× the data you pushed               ← that IS the ×T write amp
```

You rewrite a **slice**, exactly as I said in turn 1. But the slice in the lower level is `T` times bigger than the data you pushed, because the lower level is `T` times denser over that same key range. That factor of `T`, not "rewriting the whole level," is where write amplification comes from. Push 256 MB, rewrite ~2.5 GB of overlap, leave the other 390-odd boxes of L2 untouched.

## Why lower = bigger in the first place

Two forces:

**Data flows down and settles at the bottom.** New writes enter at the top (memtable → L0 → L1). Every byte eventually gets compacted downward until it reaches the last level. So the bottom level must hold essentially the **entire dataset**; the upper levels are just staging areas for recent writes on their way down. Bottom huge, top tiny is the natural shape.

**The multiplier bounds the number of levels.** If every level were the same size, holding `N` bytes would need `N / level_size` levels, so a read would probe that many runs. By making each level `T`x bigger, total capacity after `L` levels is the geometric sum ≈ `T^L`, so you need only `log_T(N)` levels:

```
equal-size levels:   #levels = N / S          (grows linearly → many runs → bad reads)
×T levels:           #levels = log_T(N)        (grows slowly   → few runs  → good reads)
```

Few levels means few runs to probe (read amp) and a bottom-heavy tree (space amp ≈ 1.1x, from the geometry we computed earlier). The price you pay for that geometry is the `×T` write amp per level. Same tradeoff triangle, now grounded in why the sizes step up.

## User

I don't understand the "The multiplier bounds the number of levels" section. 

explain in more layman and elaborate. Include concrete example with numbers and key range example

## Assistant

The claim in one line: **a read has to look in every level, so the number of levels IS the cost of a read. Growing each level 10x keeps that number tiny.** Let me build it up.

## The rule that makes level-count matter

In leveled compaction, each level is one sorted, non-overlapping run. So to find a key, you check **one SST per level** (binary search picks the single SST whose range could hold the key; bloom filter then says yes/no).

```
read cost  ≈  number of levels     (one probe per level)
```

Fewer levels = faster reads. That is the only thing to optimize here. Now, how you arrange the data into levels decides how many levels you get. Same 100 GB dataset, two ways to slice it:

## Layout A: every level the same size (100 MB each)

```
100 GB of data ÷ 100 MB per level = 1000 levels
```

One thousand levels. A lookup walks 1000 levels, probing up to 1000 SSTs.

## Layout B: each level 10x the one above

```
L1 = 100 MB
L2 = 1 GB      (×10)
L3 = 10 GB     (×10)
L4 = 100 GB    (×10)
------------------------
total ≈ 111 GB, holds the whole 100 GB in just 4 levels
```

Four levels. A lookup walks 4 levels, probing 4 SSTs.

**Same data. 1000 probes vs 4 probes.** That is what "the multiplier bounds the number of levels" means.

## Why 10x wins so hard: digits vs tally marks

Writing the number one thousand two ways:

```
tally marks (each mark = +1):   | | | | | | | | ... (1000 marks)   ← equal levels
digits    (each place = ×10):   1 0 0 0            (4 symbols)     ← geometric levels
```

Each digit position is worth 10x the one to its right, so four positions already reach 1000. Levels work identically: each level is worth 10x the one above, so four levels already reach `100 MB × 10 × 10 × 10 = 100 GB`. The formula:

```
number of levels ≈ log_T( total data / L1 size )

  geometric:  log_10(100 GB / 100 MB) = log_10(1000) = 3   → ~4 levels
  equal:      100 GB / 100 MB         = 1000              → 1000 levels
```

`log` grows painfully slowly, which is exactly what you want. Watch what happens when the database grows 10x, to 1 TB:

```
geometric:  add ONE level (L5 = 1 TB)          → 5 levels total
equal:      add 9000 levels                     → 10,000 levels total
```

10x more data costs the geometric layout a single extra probe. The equal layout falls off a cliff.

## Concrete key-range trace: look up "mango"

Geometric layout, following "mango" (starts with `m`) down. Each level's SSTs are sorted and disjoint, so binary search lands on exactly one:

```
L1 (100 MB):  [a–f] [g–n] [o–z]                       mango ∈ [g–n]   probe 1 SST, bloom: no
L2 (1 GB):    [a–c][d–f][g–i][j–n][o–z]               mango ∈ [j–n]   probe 1 SST, bloom: no
L3 (10 GB):   [a–b] ... [m–n] ... [y–z]               mango ∈ [m–n]   probe 1 SST, bloom: no
L4 (100 GB):  [aa–ac] ... [man–mao] ... [zy–zz]       mango ∈ [man–mao] probe 1 SST → FOUND

total: 4 SST probes (check newest level first; stop when found)
```

The exact same "mango" in the equal layout: 1000 levels, so up to 1000 one-SST probes for the same answer.

Notice the ranges get **narrower** as you descend, because there are more SSTs covering the same alphabet. At L1 one SST covers `g–n`; at L4 an SST covers only `man–mao`. That is the "lower level is denser" fact from before, seen from the read side: more, finer boxes, but you still touch just one per level.

(Bloom filters make the *disk* reads ≈ 1 in both layouts, since bloom says "not here" cheaply. But you still pay 1000 in-memory bloom checks vs 4, and range scans, which cannot use bloom, must merge all 1000 runs vs 4. So the level count still directly sets read work.)

## Why you tolerate the write cost

This is the payoff for the `×T` write amplification from the last turn. Each byte gets rewritten ~`T` times per level on its way down, which is the price. In exchange, the whole dataset collapses into `log_T(N)` levels, so reads and space stay cheap. That trade, expensive writes bought log-few levels, is the entire reason leveled compaction picks a multiplier instead of equal-size levels.

## User

I mean, there's no limitation that says you cannot have 1 level = 100GB ? Just keep compacting L0 to more granular, non overlapping ranges? 

is that basically become a tiered ? Or i got it wrong?

let's make the same example with tiered compaction and see where those differ?

## Assistant

You got it exactly backwards, and the backwards-ness is the most useful thing to see. Your "one 100 GB level of non-overlapping SSTs" is **not** tiered. It is the opposite of tiered. Let me place all three on one spectrum.

## First: yes, "1 level = 100 GB" is legal. It's just the read-optimal extreme.

There's no rule forbidding it. A single sorted run of 100 GB (made of bounded, disjoint SSTs) is a real, valid design. It's what you get after a "major compaction" and it's what read-only / bulk-loaded databases use. Reads are as good as it gets: **one probe, ever**, because there's one run.

It dies on **writes**, and the reason is the key-range picture. L0 flushes are memtable dumps, and random writes scatter keys across the **whole** alphabet:

```
Single level (1 sorted run), 100 GB:
  [aa-ac][ac-af]...[man-mao]...[zy-zz]      ← 400 disjoint SSTs

New L0 flush (100 MB): keys = {apple, mango, zebra, cat, ...}   scattered a..z

merge:  apple → hits [aa-ac]
        cat   → hits [cat-cau]
        mango → hits [man-mao]
        zebra → hits [zy-zz]
        ...the flush overlaps essentially ALL 400 SSTs
  ⟹ rewrite all 100 GB to absorb 100 MB       write amp ≈ 1000×
```

Every tiny flush touches the whole keyspace, so "only rewrite the overlapping slice" degenerates to "rewrite everything." **That is exactly why the intermediate ×T levels exist:** they give scattered new keys a small place to land (L1 = 100 MB), and the giant bottom level is only rewritten in bulk when an upper level fills. The multiplier is a staging buffer against this full-rewrite.

## Tiered is the write-optimal extreme: MANY overlapping runs

Tiered does not have one non-overlapping run. It keeps **several runs, each spanning the whole keyspace, overlapping each other**. Same 100 GB, ratio ~4:

```
run 5 (newest):  [ a .............. z ]   256 MB     ← each run is a FULL a..z run
run 4:           [ a .............. z ]   1 GB
run 3:           [ a .............. z ]   4 GB
run 2:           [ a .............. z ]   16 GB
run 1 (oldest):  [ a .............. z ]   64 GB
```

Look up "mango":

```
run 5: bloom → maybe → check its [man-mao]-ish SST → not here (mango not recently written)
run 4: bloom → no
run 3: bloom → no
run 2: bloom → maybe → found mango = old value → newest so far, but keep going? no,
       runs are ordered by recency, newest hit wins → stop
  ⟹ up to 5 probes, and mango may sit in MULTIPLE runs (old copy in run1, new in run5)
```

Two things differ from leveled here:
- Each run spans `a..z`, so the SST ranges do **not** narrow going down (contrast leveled, where L4's SST was just `man-mao`).
- The **same key lives in several runs** (old + new versions), because nothing deduplicates them until a merge. That's the space amplification.

Tiered's merge: whole runs merge when the count/ratio trips, rarely:

```
runs 3,4,5 pile up → ratio triggers → merge (4GB + 1GB + 256MB) into one 5.25 GB run
  each byte written ONCE this merge; runs 1,2 untouched.       write amp ≈ 5×
```

## Same 100 GB, all three side by side

```
                    reads          write amp            space on disk
                    (probes for    (rewrites of         (dup/stale
                     "mango")       a byte)              copies)
─────────────────────────────────────────────────────────────────────
1 sorted run        1   ← BEST     ~1000× ← WORST       100 GB  ← BEST
(your proposal)                    (full rewrite/flush)  (1.0×, no dups)

leveled (×10)       4             ~40× (T × levels)     ~111 GB
4 disjoint levels   (= #levels)                          (1.1×)

tiered (ratio 4)    up to 5       ~5× ← BEST            ~200 GB ← WORST
~5 overlapping runs (= #runs)                            (2×, dups linger)
```

So the three are corners/edges of the same triangle:

```
              your "1 run"  ── read-optimal, write-catastrophic  (static / read-only data)
                    │
              leveled ×T    ── the compromise                    (read/space heavy workloads)
                    │
              tiered        ── write-optimal, read/space-costly  (ingest heavy workloads)
```

Your instinct "just keep one big non-overlapping run" is the **read-optimal corner**, not tiered. Tiered is the far corner: it deliberately keeps many overlapping runs precisely so it never has to do the full rewrite that kills your single-run design. Leveled with a multiplier is the middle: enough non-overlapping levels to keep reads/space good, staged so writes don't rewrite the whole 100 GB each time.