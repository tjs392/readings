## Abstract + 1. Introduction

- **Headline:** PGM = **first learned index with provable worst-case bounds** on predecessor + range + updates. In the **static** case (queries only), bounds are **optimal**. (Theory paper that *also* wins experiments — contrast FITing-Tree's engineering framing.)
- **Unifying reduction — everything is `rank(x)`** (= # keys < x): `member` → `A[rank]==x`; `predecessor` → `A[rank−1]`; `range` → scan from `rank(x)`. So the whole paper = "compute `rank` fast + small." (`rank` = position-in-sorted-array = Kraska's CDF·N.)
- **Geometric reframing (→ the "G" in PGM):** view each key as a point **`(k, rank(k))`** in the plane; fit **lines** to them. Toy case: consecutive integers → `rank(k) = k − a`, **one line**, constant space for any `n`. Real data = piecewise-linear stretches. (= FITing-Tree's segments, framed geometrically.)
- **Why traditional indexes lose:** hash → no predecessor/range; bitmap → costly; trie → uncompressed. B-trees win by default for ordered queries → learned indexes then beat B-trees.

### Positioning vs. predecessors (= PGM's openings)
- **RMI (Kraska):** DAG of models + final binary search. Knock: tradeoffs **hard to control**, depend on data/DAG/model, **no guarantees**, needs tuning.
- **FITing-Tree:** credited for the `ε` window param. **Two criticisms = PGM's pitch:** (1) never compared vs. RMI; (2) **segmentation is sub-optimal in theory + inefficient in practice** → more segments → taller B+-tree → more space + slower. [Direct shot at ShrinkingCone's greedy non-optimality.]
- **Crux:** PGM = FITing-Tree's `ε`-bounded piecewise-linear idea, but with **optimal segmentation** + a **fully-learned recursive structure** (no B+-tree on top).

### 4 contributions (base + 3 variants)
1. **Core PGM:** optimal # of linear models for fixed `ε`, **recursive structure**; **I/O-optimal** for predecessor (by lower bound [32]); **fully learned** (RMI + FIT mix traditional + learned).
2. **Compressed (§4):** exploits repetitiveness *in the models themselves*.
3. **Distribution-aware (§5):** adapts to the **query/workload** distribution (not just data) — a first.
4. **Multicriteria auto-tuning (§6):** auto-tunes in seconds over 100Ms of keys to space/time constraints (PGM's answer to FIT's cost model, but automatic).

### Numbers to remember
- vs **FITing-Tree:** up to **75% less space**, ≥ query time.
- vs **B+-tree static:** match speed in **83× less space**.
- vs **B+-tree dynamic:** up to **71% better** query+update time, **1140× less space** (GB → MB).
- vs **RMI:** uniformly better time+space, **15× faster build, no hyperparameter tuning**.




# PGM-Index — Section 2 Notes (the static index)

*Ferragina & Vinciguerra, PVLDB 2020. Covers §2, §2.1 (optimal PLA), §2.2 (recursive index + Theorem 1).*

> **Companion figures** (in your outputs folder):
> - `rmi-vs-fiting-vs-pgm.svg` — three-generation architecture comparison (read this first).
> - `segmentation-cone-vs-hull.svg` — single-segment mechanism: ShrinkingCone vs convex hull.
> - `segmentation-multi.svg` — multi-segment, **verified against the real C++** (ε=1: 5 cones vs 4 hulls).
> - `fiting-tree-vs-pgm.svg` — how a query traverses each structure + node anatomy.
> - `pgm-recursive-space.svg` — the recursive levels and the Θ(m) geometric-series proof.

---

## The One Big Idea

**Everything is `rank`; approximate it with the *fewest possible* ε-bounded lines; then index those lines with more of the same, recursively, until one line is left.** PGM = FITing-Tree's piecewise-linear idea made (a) **optimal** in the number of segments and (b) **fully learned** (no B+-tree anywhere) — which together earn it **provable, in fact I/O-optimal, worst-case bounds**.

Two ingredients, one theorem:

| Ingredient | What it is | Section |
|---|---|---|
| **Optimal PLA-model** | the *minimum* number of ε-segments, via a convex-hull streaming algorithm | §2.1 |
| **Recursive structure** | segments indexing segments, top to bottom (no B+-tree) | §2.2 |
| **Theorem 1** | space Θ(m), query O(log m + log ε), provably I/O-optimal | §2.2 |

---

## §2 setup — the formal frame

- **Input:** a multiset `S` of `n` keys from a universe `U`, stored sorted in array `A`. *Multiset* = duplicates allowed (real columns have them). *Universe* = all possible keys, not just stored ones — a segment ε-approximates **all of `U`**, which is what lets PGM answer queries for keys that aren't present.
- **Parametric in integer `ε ≥ 1`:** `ε` is the tunable knob (same space↔speed dial as FITing-Tree). **It's an integer** (`size_t` in the code) — `ε = 1.5` is not a valid PGM parameter. `ε = 0` is pointless (no approximation → one segment per key).
- **Everything reduces to `rank(x)`** = number of keys `< x` = position of `x` in `A`. Then `member`, `predecessor`, `range` all fall out of `rank` over the sorted array. So the whole job is: **compute `rank` fast and small.**
- **Geometric reframing (the "G" in PGM):** view each key as a point `(k, rank(k))` in the plane; approximating `rank` = fitting lines to those points. A perfectly linear run (`rank(k)=k−a`) collapses to **one line** regardless of `n` — the extreme case of the compression on offer.

---

## §2.1 — The optimal PLA-model

### Definitions
- **Segment** = triple **`(key, slope, intercept)`** defining `fs(k) = k·slope + intercept`. Each is **O(1) space (two floats + one key)** and **O(1) query**. (Confirmed in `piecewise_linear_model.hpp`.)
- **ε-approximate (Def 1):** `|fs(x) − rank(x)| ≤ ε` for *all* `x` in the segment's range. **The error is vertical** — measured in position units. The source proves it: each point enters as the pair `(x, y+ε)` and `(x, y−ε)` (same `x`), so the corridor is bounded **vertically by 2ε**. (The paper's "rotated rectangle of height 2ε" = a *tilted band whose vertical gap is 2ε*; the rigorous Def 1 and Lemma 2 are vertical.)
- **Optimal PLA-model (Def 2):** the **minimum number** of ε-segments with disjoint ranges covering `U`. ← the explicit upgrade over FITing-Tree's "some greedy number."

### How it's computed — convex hull (Lemma 1)
*See `segmentation-cone-vs-hull.svg`.*
- DP solves it in O(n³) — too slow. PGM **imports** a known **O(n) optimal streaming** algorithm from the time-series / lossy-compression literature.
- **The reduction:** "can one line ε-approximate these points?" ⟺ "do they fit in a vertical **2ε-thick band** (of some slope)?" ⟺ "does their **convex hull** fit in a 2ε-tall (tilted) rectangle?" Whether a 2ε band fits depends only on the **extreme points = the hull**, so you stream points, maintain the hull, and cut the instant it no longer fits.
- **In the code:** keeps `upper` (hull of the `y+ε` points) and `lower` (hull of the `y−ε` points); `rectangle[0..3]` are the 4 points defining the corridor's two extreme feasible lines. Slope chosen = **midpoint of the feasible slope range**; **intercept freely computed** → the line is *not pinned to any point*. All arithmetic is exact integer (cross-products, `__int128`) so the ε-bound holds rigorously, not "within rounding."

### The load-bearing bound — Lemma 2
- **Claim:** every optimal segment covers **≥ 2ε keys**, so the number of segments **`m ≤ n/(2ε)`**.
- **Proof (one line):** for any `2ε` consecutive keys, their y-values are the consecutive integers `i, …, i+2ε−1`; the flat line `y = i+ε` is within `ε` of all of them → a valid segment *always* exists for any `2ε`-run → the optimal algorithm is never forced to cut before `2ε` points. ∎
- **Why it matters:** foundation of *every* bound in Theorem 1 (size, levels, optimality). PGM's analog of FITing-Tree's Theorem 3.1 (`≥ ε+1`), and slightly stronger thanks to the symmetric `2ε` band.

### Why optimal beats greedy-cone (the proof sketch)
*See `segmentation-multi.svg` — at ε=1, cone = 5 segments, hull = 4.*
- **Greedy "extend each segment maximally" is provably optimal** via an **exchange / stay-ahead argument:**
  - Let `R(a)` = furthest endpoint reachable from start `a`. Validity is **prefix-monotone** (sub-ranges of a valid segment are valid) and **monotone in the start** (`a'≥a ⇒ R(a')≥R(a)`).
  - Induction: greedy's `j`-th boundary `g_j ≥ o_j` (any optimal segmentation's `j`-th boundary). Base case: both start at 0, and OPT's first segment can't exceed `R(0)`. Step: greedy starts segment `j+1` at `g_j ≥ o_j`, so by monotonicity reaches `R(g_j) ≥ R(o_j) ≥` OPT's next boundary.
  - "Never behind at any step" ⇒ greedy finishes in ≤ OPT's segment count ⇒ **greedy = optimal.**
- **The convex hull is what makes each segment *truly* maximal** (computes the real `R(a)`, fast) — the exact hypothesis the argument needs.
- **Why ShrinkingCone fails this proof:** it pins the line through the segment's **first point** (fixes the *intercept*), optimizing only the **slope**. So it computes `R_cone(a) ≤ R(a)` over a *handicapped* line set; the base case already breaks (`o_1−1 ≤ R(0)` but maybe `> R_cone(0)`), and "stay ahead" collapses. One fixed degree of freedom = the whole optimality gap. **Same O(n) cost, strictly better result** — which is why the paper calls FITing-Tree's segmentation "sub-optimal in theory and inefficient in practice."

---

## §2.2 — Indexing the PLA-model (the recursive structure)

*See `fiting-tree-vs-pgm.svg` for traversal + node anatomy, `pgm-recursive-space.svg` for the levels.*

### The problem and the move
- Given `m` segments, a query must find the **rightmost segment with `key ≤ k`**. Option A: index them with a B+-tree (= FITing-Tree) → `O(log_B m + log(ε/B))` I/Os, **but fixed fan-out is blind to the key distribution.**
- **PGM's move:** "indexing segments" is the *same problem* as "indexing data" → solve it with the same tool. Take the **first key of each segment** → build another optimal PLA-model over them → repeat (each level ≥ `2ε` smaller) → **one root segment**.
- **Result = a "pure / fully learned" index:** no B+-tree, no fallback. Contrast RMI (B-tree fallback) and FITing-Tree (B+-tree on top) — both *mix* learned and traditional. *(See the `rmi-vs-fiting-vs-pgm.svg` matrix.)*

### Three advantages over a B+-tree
1. **Variable fan-out** = keys-per-segment (data-adaptive; smooth data → huge fan-out → very few levels), not a fixed page size.
2. Each node's segment is an **O(1)-space, O(1)-time ε-approximate routing table** — evaluate one line to know roughly where to go (a B+-tree node's search cost grows with node size).
3. The per-node correcting search is **`log ε`**, *independent of how many keys the segment covers.*

### The query (matches Figure 2 / the traversal SVG)
At each level: **evaluate the segment's line → estimate the position in the level below → binary-search a `2ε` window there → that identifies the next segment → descend.** Repeat to the bottom array. (Footnote-4 patch: for a key falling *between* two segments, use `min{fs_j(k), fs_{j+1}(s_{j+1}.key)}` to stay bounded.)

### Node anatomy — the deep difference from FITing-Tree
*(detailed in `fiting-tree-vs-pgm.svg`)*
- **FITing-Tree** has **three** node types: B+-tree inner node `{separator keys, child ptrs}`, B+-tree leaf `{segment first-keys, ptrs}`, and a separate segment object `{startKey, slope, dataPtr, buffer}`. Routing = **compare key vs. separators**, follow a **stored pointer**.
- **PGM** has **one** node type — the segment `(key, slope, intercept)` — serving as root, internal, and leaf alike. Routing = **evaluate `slope·k + intercept`**, then 2ε-search. **No child pointers, no separators** — navigation is *computed*, not stored. That uniformity is *why* it's "fully learned," and a big reason it's smaller (a B+-tree spends real bytes on separators + pointers at every level).

---

## Theorem 1 — the bounds and their proofs

> **Statement.** PGM with parameter `ε` indexes `A` in **Θ(m) space**, answers rank/membership/predecessor in **O(log m + log ε)** time and **O((log_c m)·log(ε/B))** I/Os, where `m` = min #segments, `c ≥ 2ε` = variable fan-out, `B` = block size. Range queries add **O(K)** time / **O(K/B)** I/Os for `K` results.

Everything below rests on **Lemma 2** (`≤ N/(2ε)` segments per level). *Visualized in `pgm-recursive-space.svg`.*

**(1) Levels `L = O(log_c m)`.** Counts shrink by ≥ `2ε` each level: `m → m/(2ε) → m/(2ε)² → … → 1`. Collapsing `m` to `1` by dividing by ≥ `2ε` takes `log_{2ε} m` levels (with real fan-out `c`, `log_c m`). In practice `c ≫ 2ε`, so `L` is tiny (2–4 levels for hundreds of millions of keys).

**(2) Space Θ(m).** Sum all levels:
```
Σ_{ℓ=0..L}  m/(2ε)^ℓ  =  m · Σ (1/2ε)^ℓ  ≤  m · 1/(1 − 1/2ε)  =  Θ(m)
```
A **geometric series with ratio 1/(2ε) ≤ ½** → converges to a constant × first term. So *all recursive levels combined* are just a constant tax on top of the `m` leaf segments. Each segment is O(1) → total **Θ(m)**. (Numbers: `m=64, 2ε=4` → `64+16+4+1 = 85 ≈ 1.33m`.)

**(3) Query time O(log m + log ε).** `L` levels, each a binary search over a `2ε` window (`log 2ε` each). At worst case `c = 2ε`:
```
L · log(2ε) = log_{2ε}(m) · log(2ε) = log₂ m     (change-of-base cancels)
```
plus the final bottom search `log ε` → **O(log m + log ε)**. Intuition: **descent costs `log m`, last-mile correction costs `log ε`.**

**(4) I/O O((log_c m)·log(ε/B)).** Same `L` levels; each `2ε`-window search costs `log(ε/B)` I/Os (a block holds `B` keys, so the window spans `ε/B` blocks).

**(5) Range queries.** One `rank` query to find the start, then a **sequential scan** of the `K` matches: **O(K)** time, **O(K/B)** I/Os — optimal, since you must at least read the `K` outputs.

---

## The two payoffs (remember these)

1. **Space depends on *regularity*, not `n`.** Since `m ≤ n/(2ε)` at *every* level, size `Θ(m)` is governed by how regular the data is, not the raw key count. Setting `c = 2ε = Θ(B)` in Theorem 1 recovers a `2ε`-way tree, so **PGM is never asymptotically worse than a B+-tree / FITing-Tree / CSS-tree** — the graceful floor — and usually far better (`m ≪ n`).
2. **Provably I/O-optimal.** By the lower bound from [Patrascu–Thorup, ref 32], no structure solves predecessor search with fewer I/Os. So PGM **can replace any index "with virtually no performance degradation."** Strongest possible claim: not "faster in our tests" but "you can't asymptotically beat this."
- *Practical kicker:* real data gives `m ≪ n` and `c ≫ 2ε`, so it beats even the worst-case bounds → §7 experiments.

---

## How PGM differs from RMI and FITing-Tree (consolidated)

*Full matrix in `rmi-vs-fiting-vs-pgm.svg`. The arc in one line: RMI proved learning beats B-trees but gave **no guarantees**; FITing-Tree added a **hard ε + updates** atop a B+-tree; PGM made segmentation **optimal** and the structure **fully learned**, earning **provable bounds**.*

| | **RMI (Kraska)** | **FITing-Tree** | **PGM** |
|---|---|---|---|
| Model | DAG of ML models (can be NNs) | linear segments | linear segments |
| Error | **measured after training** (no bound) | ε **at construction** (hard) | ε at construction (hard) |
| Segmentation | n/a (train models) | **ShrinkingCone** (greedy, near-optimal, *pinned*) | **convex hull** (greedy, **optimal**, *floating*) |
| Routing | model picks next model + **B-tree fallback** | **B+-tree** over segments | **recursive segments** (none) |
| Fully learned? | ✗ (fallback) | ✗ (B+-tree) | **✓** |
| Updates | ✗ read-only | ✓ delta buffer | ✓ LSM levels (§3) |
| Worst-case | **none** | bounded lookup | **provably I/O-optimal** |
| Build | slow, needs hyperparameter tuning | fast | **15× faster than RMI, no tuning** |

**Three precise upgrades PGM makes over FITing-Tree:**
1. **Optimal vs. near-optimal segmentation** — convex hull (free line) instead of ShrinkingCone (pinned line). Fewer segments → smaller, shorter index. *(The pin = one fixed degree of freedom = the whole gap; see `segmentation-multi.svg`.)*
2. **Recursive segments vs. B+-tree on top** — fully learned, computed navigation, no separators/pointers. *(see `fiting-tree-vs-pgm.svg`.)*
3. **Provable bounds vs. empirical performance** — Θ(m) space, O(log m + log ε) query, I/O-optimal.

---

## Quick-reference cheat sheet

```
rank(x)          = #keys < x = position in A  (everything reduces to this)
segment          = (key, slope, intercept);  fs(k)=slope·k+intercept
ε-approximate    : |fs(x) − rank(x)| ≤ ε      (VERTICAL error, ε is an integer)
optimal PLA      : min #segments; convex-hull streaming, O(n) time/space (Lemma 1)
Lemma 2          : each segment ≥ 2ε keys  ⇒  m ≤ n/(2ε)
recursive index  : first-key of each segment → new PLA-model → … → 1 root (fully learned)
node             : one triple; route by COMPUTING fs(k) + 2ε binary search (no pointers)
Theorem 1        : space Θ(m); time O(log m + log ε); I/O O(log_c m · log(ε/B)); range +O(K)
why Θ(m)         : Σ m/(2ε)^ℓ = geometric series, ratio ≤ ½ → Θ(m)
why optimal      : greedy stay-ahead + hull computes true R(a); cone pins intercept → loses it
payoffs          : space ∝ regularity not n; never worse than B+-tree; provably I/O-optimal
```



