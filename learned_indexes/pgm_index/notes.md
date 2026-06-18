# PGM-Index — Complete Study Notes

*Ferragina & Vinciguerra, "The PGM-index: a fully-dynamic compressed learned index with provable worst-case bounds," PVLDB 13(8), 2020.*
*Reading lineage: Kraska RMI (2018) → FITing-Tree (2019) → **PGM (here)** → ALEX (next).*

---

## Orientation — what this paper is

PGM is the **first learned index with provable worst-case bounds** on predecessor + range + updates, and in the **static** (read-only) case those bounds are **optimal**. It's a theory paper that *also* wins the experiments — the opposite emphasis from FITing-Tree's engineering framing.

The whole paper hangs on one reduction and one structure:
- **Reduction:** every operation is `rank(x)` (= #keys `< x` = position in the sorted array). `member` → `A[rank]==x`; `predecessor` → `A[rank−1]`; `range` → scan from `rank`. So the entire job is *compute `rank` fast and small*.
- **Structure:** approximate `rank` with the **fewest possible** ε-bounded lines (segments), then index those segments with **more of the same**, recursively, until one line is left. No B+-tree anywhere = "fully learned."

Everything else is built on that static **atom**: §3 makes it updatable (a shelf of atoms), §4 compresses one atom, §5 builds one atom shaped by the query distribution, §6 auto-picks the atom's ε.

### Figure index (companion SVGs)
| File | Shows |
|---|---|
| `rmi-vs-fiting-vs-pgm.svg` | three-generation architecture comparison + feature matrix |
| `segmentation-cone-vs-hull.svg` | single segment: ShrinkingCone (pinned) vs convex hull (floating) |
| `segmentation-multi.svg` | multi-segment, **verified vs real C++** (ε=1: 5 cones vs 4 hulls) |
| `fiting-tree-vs-pgm.svg` | query traversal of each structure + node anatomy |
| `pgm-recursive-space.svg` | recursive levels + the Θ(m) geometric-series proof |
| `dynamic-pgm-structure.svg` | runs = separate whole PGMs; insert/merge, delete, GC |
| `dynamic-pgm-cascade.svg` | inserts as a binary counter + anatomy of a carry |
| `dynamic-pgm-reads.svg` | point + range query across runs (heap-merge) |
| `dynamic-pgm-query.svg` | tombstone shadowing + the LSM bargain + ALEX contrast |
| `compressed-pgm.svg` | slope interval-stabbing + intercept compression |
| `distribution-aware-pgm.svg` | per-key tolerance, search windows, telescoping, entropy |
| `pgm-master-map.svg` | full data-flow of every algorithm and when it runs |

### Numbers to remember
- vs **FITing-Tree:** up to **75% less space**, ≥ query speed.
- vs **B+-tree (static):** match speed in **83× less space**; one case **4 orders** less (874 MiB → 87 KiB).
- vs **B+-tree (dynamic):** up to **71% better** query+update time, **611–3891× less space** (GB → MB).
- vs **RMI:** uniformly better time+space, **14.5× faster build**, **no hyperparameter tuning**, far more predictable latency.

---

## §1 — Positioning (= PGM's openings on its predecessors)

- **RMI (Kraska):** DAG of models + final binary search. Knock: tradeoffs **hard to control**, depend on data/DAG/model, **no guarantees**, needs tuning.
- **FITing-Tree:** credited for the `ε`-window idea. Two criticisms = PGM's pitch: (1) never compared vs RMI; (2) **segmentation is sub-optimal in theory + inefficient in practice** → more segments → taller B+-tree → more space + slower. (A direct shot at ShrinkingCone's greedy non-optimality.)
- **Crux:** PGM = FITing-Tree's ε-bounded piecewise-linear idea, but with **optimal segmentation** + a **fully-learned recursive structure** (no B+-tree on top) → **provable, I/O-optimal bounds**.
- **Why traditional indexes lose:** hash → no predecessor/range; bitmap → costly; trie → uncompressed; B-trees win by default for ordered queries, so learned indexes aim to beat B-trees.

**Four contributions:** (1) core PGM — optimal #models + recursive structure, I/O-optimal, fully learned; (2) **compressed** (§4); (3) **distribution-aware** (§5, a first); (4) **multicriteria auto-tuning** (§6).

---

## §2 — The Static Index (the atom)

> Figures: `rmi-vs-fiting-vs-pgm.svg`, `segmentation-cone-vs-hull.svg`, `segmentation-multi.svg`, `fiting-tree-vs-pgm.svg`, `pgm-recursive-space.svg`.

**One big idea:** everything is `rank`; approximate it with the *fewest* ε-bounded lines; index those lines with more of the same, recursively, to one root. Optimal **and** fully learned → provable bounds.

### Setup
- Input: a **multiset** `S` of `n` keys from universe `U`, sorted in array `A`. A segment ε-approximates **all of `U`** (not just stored keys) — that's what lets PGM answer absent-key queries.
- **`ε ≥ 1` is an integer** (`size_t` in the code; `ε=1.5` invalid). Same space↔speed dial as FITing-Tree.
- Geometric reframing (the "G"): each key is a point `(k, rank(k))`; approximating `rank` = fitting lines. A perfectly linear run collapses to **one line** regardless of `n`.

### §2.1 — The optimal PLA-model
- **Segment** = `(key, slope, intercept)`, `fs(k)=k·slope+intercept`. **O(1) space (2 floats + key), O(1) eval.**
- **ε-approximate (Def 1):** `|fs(x) − rank(x)| ≤ ε` for all `x` in range. **Error is VERTICAL** (position units): each point enters as `(x, y+ε)` and `(x, y−ε)`, so the corridor is bounded vertically by `2ε`. (The paper's "rotated rectangle of height 2ε" = a tilted band whose vertical gap is 2ε; Def 1 / Lemma 2 are rigorously vertical.)
- **Optimal PLA (Def 2):** the **minimum** number of ε-segments covering `U` — the explicit upgrade over FITing-Tree's "some greedy number."
- **Computed via convex hull (Lemma 1):** DP is O(n³); PGM imports a known **O(n) optimal streaming** algorithm. Reduction: "can one line ε-approximate these points?" ⟺ "do they fit a vertical 2ε band of some slope?" ⟺ "does their **convex hull** fit a 2ε-tall tilted rectangle?" Stream points, maintain the hull, cut the instant it no longer fits. In code: keep `upper`/`lower` hulls of the `y±ε` points; chosen slope = **midpoint of feasible slope range**; **intercept freely computed** (line *not pinned to any point*); exact integer arithmetic so the bound is rigorous.

### Lemma 2 (load-bearing)
- Every optimal segment covers **≥ 2ε keys** → **`m ≤ n/(2ε)`**.
- One-line proof: for any `2ε` consecutive keys, the y-values are consecutive integers `i…i+2ε−1`; the flat line `y=i+ε` is within ε of all → a valid segment always exists for any `2ε`-run → never forced to cut before `2ε` points. ∎
- Foundation of **every** Theorem-1 bound. (Analog of FITing-Tree's Thm 3.1 `≥ε+1`, slightly stronger via the symmetric band.)

### Why optimal beats greedy-cone
> `segmentation-multi.svg`: at ε=1, cone = 5 segments, hull = 4.

Greedy "extend each segment maximally" is **optimal** via a stay-ahead argument: validity is prefix-monotone and monotone in the start (`a'≥a ⇒ R(a')≥R(a)`), so by induction greedy's j-th boundary `g_j ≥ o_j` (any optimal's), hence greedy finishes in ≤ OPT's count. **The convex hull computes the true furthest reach `R(a)`**, which is the hypothesis the argument needs. **ShrinkingCone fails it** because it pins the line through the segment's first point (fixes the intercept, optimizes only slope) → computes `R_cone(a) ≤ R(a)` over a handicapped line set → the stay-ahead base case already breaks. **One fixed degree of freedom = the entire optimality gap. Same O(n) cost, strictly better result.**

### §2.2 — The recursive structure
- "Indexing segments" = "indexing data" → use the same tool. Take the **first key of each segment** → build another optimal PLA over them → repeat (each level ≥ `2ε` smaller) → **one root**. No B+-tree = **fully learned** (contrast RMI's B-tree fallback, FITing-Tree's B+-tree on top — both *mix* learned + traditional).
- **Node = one segment**, serving as root/internal/leaf alike. Routing = **evaluate `slope·k + intercept`**, then binary-search a `2ε` window → descend. **No separators, no child pointers — navigation is computed, not stored.** (FITing-Tree has three node types and stored pointers; this uniformity is *why* PGM is "fully learned" and a big reason it's smaller.)
- Three edges over a B+-tree: **variable fan-out** (= keys-per-segment, data-adaptive); each node an **O(1) routing table**; per-node correction is **`log ε`**, independent of segment size.

### Theorem 1
> Space **Θ(m)**, query **O(log m + log ε)** time, **O((log_c m)·log(ε/B))** I/Os; `c ≥ 2ε` = variable fan-out; range adds **O(K)** / **O(K/B)**.

Proofs (all via Lemma 2):
- **Levels `L = O(log_c m)`** — counts shrink by ≥ `2ε`/level; in practice `c ≫ 2ε` → 2–4 levels for 100Ms of keys.
- **Space Θ(m)** — `Σ m/(2ε)^ℓ` is a geometric series, ratio ≤ ½ → converges to constant × `m`. (e.g. `m=64, 2ε=4` → `64+16+4+1=85≈1.33m`.) All levels = a constant tax on the leaf segments.
- **Query O(log m + log ε)** — `L · log(2ε) = log₂ m` (change-of-base cancels) + final `log ε`. **Descent costs `log m`, last-mile correction costs `log ε`.**
- **I/O** — each `2ε`-window search spans `ε/B` blocks → `log(ε/B)` I/Os per level.
- **Range** — one `rank` + sequential scan of K → O(K) / O(K/B), optimal.

### Two payoffs
1. **Space ∝ regularity, not `n`** (`m ≤ n/2ε` at every level). Setting `c=2ε=Θ(B)` recovers a `2ε`-way tree → **never asymptotically worse than a B+-tree** (graceful floor), usually far better.
2. **Provably I/O-optimal** (Patrăscu–Thorup lower bound) → PGM can replace any index "with virtually no performance degradation." Not "faster in our tests" but "you can't asymptotically beat this."

### Cheat sheet — §2
```
rank(x)        = #keys < x = position in A  (everything reduces to this)
segment        = (key, slope, intercept); fs(k)=slope·k+intercept; O(1) space/eval
ε-approx       : |fs(x)−rank(x)| ≤ ε   (VERTICAL error, ε integer)
optimal PLA    : min #segments; convex-hull streaming O(n) (Lemma 1); slope=midpoint, intercept free
Lemma 2        : each segment ≥ 2ε keys ⇒ m ≤ n/(2ε)
recursive index: first-key of each segment → new PLA → … → 1 root (fully learned, computed routing)
Theorem 1      : space Θ(m); time O(log m+log ε); I/O O(log_c m·log(ε/B)); range +O(K)
why Θ(m)       : Σ m/(2ε)^ℓ geometric, ratio ≤ ½ → Θ(m)
why optimal    : stay-ahead + hull computes true R(a); cone pins intercept → loses it
payoffs        : space ∝ regularity not n; never worse than B+-tree; provably I/O-optimal
```

---

## §3 — The Dynamic Index (inserts/deletes)

> Figures: `dynamic-pgm-structure.svg` (read first), `dynamic-pgm-cascade.svg`, `dynamic-pgm-reads.svg`, `dynamic-pgm-query.svg`.

**One big idea:** the dynamic PGM is **not one index — it's a shelf of complete static PGMs of sizes 1, 2, 4, 8, …**; updates never edit a run in place, they **rebuild runs by merging**. Inserts = incrementing a binary counter; deletes = tombstones; reads pay by visiting every run.

### Why updates are hard for PGM
- A B-tree node holds a **fixed Θ(B) keys** → split/merge by counting. A PGM segment indexes a **variable, huge** count → no trigger; classic algorithms inapplicable.
- A segment is a **line predicting positions**, not a container: inserting mid-array shifts every later key's position → the line is wrong for thousands of keys → recompute cascades up levels. One insert can invalidate a chunk.
- Forcing fixed-size segments would fix that but **"disintegrates the indexing power"** (collapses toward a B-tree). And it rejects the per-node buffer+retrain fix (FITing-Tree/RMI): slow for big models, thrashes under skew.

### Strategy 1 — Append-only (time series)
Keys arrive sorted at the end → no key shifts. Try to **extend the last segment** within ε (the streaming hull just continues, O(1) amortized); else start a new segment and recurse to the last segment one level up, until absorbed or the root splits. **O(log_c m) amortized.**

### Strategy 2 — Arbitrary inserts: the logarithmic / Bentley–Saxe method
*(= the `DynamicPGMIndex` code.)*
- **Structure:** independent static PGMs over sets `S₀…S_b`, each **empty or exactly 2ⁱ keys**, `b=Θ(log n)`. **Each `Sᵢ` is a complete §2 PGM** (own segments + array), side by side — not a leaf/level inside one index. (Small runs skip the index, just binary-searched.) Two meanings of "level": recursive levels *inside* a run vs. the shelf of runs *across* the structure.
- **Insert = carry:** find first empty `Sᵢ`; merge `S₀∪…∪S_{i−1}∪{x}` (linear, all sorted) → build a fresh static PGM = new `Sᵢ`; empty the runs below. Works because `2ⁱ = 1 + Σ_{j<i}2ʲ` → exactly 2ⁱ keys. The full/empty pattern = the binary value of n; the insert is the carry rippling to the first empty slot (`0011+1=0100`).
- **Cost O(log n) amortized:** a key moves up ≤ `b=Θ(log n)` levels over its life, O(1) per merge. Big merges are rare (run i rebuilds once per 2ⁱ inserts).
- **Delete = tombstone (+ free GC):** `erase(k)` inserts a sentinel ⌫k (disguised insert). A tombstone in a **newer** run shadows the real key in an **older** one on reads. When those runs later merge, ⌫k and k cancel — GC for free. Everything stays append-and-merge.
- **Sorted within, overlapping across:** runs partition by **recency, not value** → key ranges overlap. Global order is never stored; it's reconstructed on read (newest-wins for points, heap-merge for ranges). Merges fold small runs into the big sorted bulk.
- **Does 2ⁱ sizing hurt variable segments? No.** The power-of-two constraint is on **run totals**, not segment counts; inside each run, segmentation is optimal and data-driven (box capacity fixed, book thickness free). Only cost: a few **seam segments** at run boundaries (∝ log n), healed by merges.

### Reading the dynamic index
- **Point/predecessor:** search **every** non-empty run (each a §2 lookup), newest-first → **O(log n·(log m+log ε))**.
- **Range:** slice each run, heap-merge (size b), drop tombstoned keys → **O(log n(log m+log ε)+K log log n)** (or +K buffered).

### Theorem 2
> Query **O(log n·(log m+log ε))**; insert/delete **O(log n) amortized**. EM: query **O((log_B n)(log_c m))** I/Os, update **O(log_B n)** amortized. Range +O(K)/O(K/B).
>
> vs static Theorem 1: every query pays one extra **log n** factor — the per-run search (= **read amplification**). De-amortizable to worst-case (≤3 copies per `Sᵢ`, spread each merge over 2ⁱ inserts), but the amortized version already beats B+-trees (§7.3).

### The LSM bargain
**Gained:** cheap, amortized, skew-tolerant writes; the optimal static index reused unchanged per rebuild. **Paid:** data across O(log n) runs → every read ×log n; merges rebuild whole runs; write latency is amortized-cheap but bursty. Good trade iff write-heavy.

### Cheat sheet — §3
```
dynamic PGM = SHELF of complete static PGMs, sizes 1,2,4,…,2^b (b=Θ(log n)); runs by RECENCY
append insert : extend last segment, recurse up → O(log_c m) amortized
arbitrary     : carry into first empty run; merge+rebuild (binary counter) → O(log n) amortized
delete        : insert TOMBSTONE; shadows key on read; GC free at next merge
point query   : search EVERY run newest-first → O(log n·(log m+log ε))
range query   : slice each + heap-merge → O(log n(log m+log ε)+K log log n)
Theorem 2     : +log n vs static = read amplification (the LSM bargain)
```

---

## §4 — The Compressed Index

> Figure: `compressed-pgm.svg`.

**One big idea:** a segment stores two numbers; compress both by exploiting structure already there for free. **Lossless, ε preserved, queried in place (no decompression).** This is a **variant** — query/update algorithms unchanged.

### §4a — Intercepts (easy)
In each segment's coordinate system, intercept = **position of its first key** → **monotone increasing**, integer, `< n`. Truncate to `⌊intercept⌋` (**+1 to ε**, bounded). `m` increasing integers `< n` → a known **succinct structure**. **Prop 1:** `m·log(n/m)+1.92m+o(m)` bits, **O(1) random access**.

### §4b — Slopes (the novel part)
- **Why a segment has many valid slopes:** ε is a *band*, not a line — a range of tilts threads every point's tolerance. §2's hull **already computed this interval** (`min/max_slope`) and stored only the midpoint. §4 stops discarding the slack.
- **Opportunity:** overlapping slope intervals can **share one slope** → cut distinct slopes `m → t ≪ m`. Store `t` floats in a table `T`; each segment keeps a **`⌈log t⌉`-bit pointer**.
- **Algorithm (greedy interval-stabbing):** sort intervals lexicographically; sweep maintaining intersection `(l,r)`; absorb while overlapping; **cut when `aj > r`**; pick a slope in `(l,r)` for the whole group; repeat. (`{(2,7),(3,6),(4,8),(7,9)}` → first 3 share `(4,6)`; `(7,9)` new.) = classic min-points-to-stab-intervals; greedy optimal. Same "stream + running feasible region + commit when it breaks" shape as segmentation.
- **Theorem 3:** minimum `t ≤ m`, ε preserved; **O(m log m)** time; **`64t + m⌈log t⌉` bits**. Win because `t ≪ m`.

### Three clarifications
1. **Why multiple valid slopes?** ε is a band; §2 computed the exact range and kept only the midpoint. The slack was always there.
2. **Does it change each node's slope? Yes — and that's why it's lossless.** Each segment adopts its group's shared slope, but that slope lies **inside its own feasible interval**, so ε holds and the ±ε search still finds the key.
3. **Decompress per access? No.** One **O(1)** table lookup for the slope, **O(1)** succinct access for the intercept; the query path is byte-for-byte identical plus one indirection. The compressed form *is* the operational form.

### Cheat sheet — §4
```
intercept: monotone int < n → succinct (Prop 1): m·log(n/m)+1.92m+o(m) bits, O(1) access (+1 ε)
slope:    each segment has an INTERVAL of valid slopes (a §2 byproduct)
          overlapping intervals SHARE a slope → t ≪ m; greedy interval-stabbing (Thm 3)
          store 64t (table) + m⌈log t⌉ (pointers); lossless, ε holds; NO decompress
```

---

## §5 — The Distribution-Aware Index

> Figure: `distribution-aware-pgm.svg`.

**One big idea:** **spend error budget inversely to query frequency.** Tolerance `yi = min(1/pi, ε)` per key → hot keys get tiny windows (fast `log(1/pi)`), cold keys keep loose ε; routing weights make per-level costs **telescope** to a single `O(log 1/pi)`. Result: **O(H) average time** (entropy floor) in **O(m) space**. The first distribution-aware learned index. (Same spirit as Huffman / optimal BSTs, via segment geometry.)

### The mechanism
- **Problem:** Theorem 1 assumes **uniform** queries; real workloads are **skewed** (Zipf). Want hot queries faster.
- **Goal:** search `ki` in **O(log 1/pi)** → average = **entropy H = Σ pi·log(1/pi)**, the optimum.
- **Core trick:** Lemma 1 extends to per-point y-ranges (still O(n)). Set `yi = min{1/pi, ε}`: frequent → small window → fast; rare → ε window → slow but rare. Final search = `O(log min{1/pi,ε}) = O(log 1/pi)`. **Space cost:** sub-ε keys add ≤ **2ε** extra segments/level → bottom Θ(m+ε) = O(m).
- **Telescoping routing:** for segment `s[a,b]`, `Pa,b=Σpi` (prob. of reaching it), `qa,b=max pi`; carry the first key up with weight **`qa,b/Pa,b`**. Per-level costs `log(Pa,b/pi)`, `log(Pa′,b′/Pa,b)`, … cancel across levels → whole path = **O(log 1/pi)**.
- **Theorem 4:** **O(m) space, O(H) average query time.** Distribution-awareness is free in space.

### Clarification — "rebuilding"? No
A **static build** (sibling of §2), **not** the §3 cascade; **one** index, not a shelf. The segmentation is just fed `yi=min(1/pi,ε)` per key, so hotness shapes the fit (tighter near hot keys) at every level — once. **Keys are not moved** by hotness; only their tolerance changes.

### Why no shelf here
The shelf (§3) exists only for **updates**. §5 is a **read** optimization on **static** data, so the shelf (and its log n read tax) would be pure cost. All three variants modify the **static atom**; the shelf is the one mechanism for *arranging many atoms*.

### Cheat sheet — §5
```
goal       : search ki in O(log 1/pi) → average = entropy H (optimum)
core trick : per-key tolerance yi = min(1/pi, ε); hot → tiny window/fast, cold → ε/slow-but-rare
space      : ≤ 2ε extra segments/level → O(m)
routing    : carry first key with weight qa,b/Pa,b → per-level costs TELESCOPE to O(log 1/pi)
Theorem 4  : O(m) space, O(H) average time. Static build, ONE index (no shelf).
```

---

## §6 — The Multicriteria Index (auto-tune ε to a budget)

**One big idea:** ε is a **single, monotone** knob (ε↑ → fewer segments → less space but wider 2ε window → more time), so tuning is a clean **1-D optimization** — "give me a budget, not a number." PGM's automatic answer to FITing-Tree's cost model.

- **Two dual problems:** (i) **min-time** given space `s_max`; (ii) **min-space** given time `t_max`.
- **Models:** time `t(ε)=δ·(log_{2ε}m)·log(2ε/B)`; space `s(ε)=Σ s_ℓ(ε) ≤ (2εm−1)/(2ε−1)` (geometric, via Lemma 2). **Catch:** no closed form for `s(ε)`, only a bound → model `m ≈ a·ε^{−b}` (power law fit per dataset; interpolates worst case ↔ linear best case).
- **Time-min solve:** fastest-that-fits = smallest affordable ε = root of `s(ε)=s_max`. Binary-search `E=[B/2, n/2]` (lower: 2ε≥B else no I/O savings; upper: 2ε≤n by Lemma 2); **speed up** by guessing the root via the power law, **biased toward the midpoint** (convex combo) → fast when the model is good, **degrades gracefully to binary search**.
- **Space-min solve + reality check:** the time *model* is unreliable on real hardware (caches, prefetch). Fix: **measure, don't model** — steer by **actual measured avg query time** over a fixed random batch; exponential search from the dominating term `c·log(2ε/B)=t_max`; **stop when the range < threshold** (don't fit measurement noise).
- **Where it sits:** closes §2's open "who picks ε?"; an early **self-tuning / instance-optimized** data structure (configures to data + workload + hardware).

### Cheat sheet — §6
```
knob       : single, MONOTONE ε (space↓, time↑) → 1-D optimization
models     : t(ε)=δ·log_{2ε}m·log(2ε/B); s(ε) ≤ (2εm−1)/(2ε−1); m ≈ a·ε^{−b} (power law)
min-time   : root-find s(ε)=s_max in [B/2,n/2]; model-guess biased to midpoint (degrades to binsearch)
min-space  : measure actual t̄(ε) (model unreliable); exponential search; stop on noise threshold
```

---

## §7–8 — Experiments & Conclusions (empirical validation)

**Setup:** real C++ impl (public repo), Xeon Gold. Datasets: Web logs (715M timestamps, biggest), Longitude (166M OSM coords), IoT (26M) + synthetic (uniform/Zipf/lognormal). The variety shows optimality holds across distributions.

**§7.1 Space — validates §2 optimality.** Optimal PLA vs greedy cone, counting segments (= space): **20–75% fewer** (synthetic 10⁹), **30–63% fewer** (real) — **for free** (both linear-time; ~2.59s Web logs @ε=8). **m is 2–5 orders of magnitude < n** (Fig 3). The core learned-index payoff, measured.

**§7.2 Query — recursive structure wins.** PGM◦REC dominates PGM◦BIN/PGM◦CSS; mechanism = **5 levels vs CSS's 7** (high 2ε branching → shorter traversal, as §2.2 conjectured); above ε=256 all converge (fits L2). vs traditional: matched CSS-tree (341 MiB) in **4 MiB (82.7×)**; matched B+-tree (874 MiB) in **87 KiB (4 orders)**. vs RMI: PGM dominates, with **MAE 226±139 vs RMI's 892±3729** — lower *and* far more predictable (guaranteed ε vs train-and-hope); RMI build 14.5× slower.

**§7.3 Dynamic — CONFIRMS the §3 skepticism ⚑.** Mixed workload, query fraction swept 0→1 (rest = inserts+deletes), Dynamic PGM vs B+-trees:
- **Write-heavy: PGM wins 13–71%** (B+-tree pays split/merge).
- **Read-heavy (0.8, 0.9): PGM LOSES** (1% / **15.2% slower**) ← the log n read amplification, *measured*.
- **Pure queries (1.0): PGM wins** (no shelf → equivalent to a static bulk-loaded PGM).
- Space: **611–3891× smaller** than the B+-trees regardless.
- → the non-monotone latency curve *is* the depth/breadth, granularity-mismatch critique rendered as a graph. Reported honestly via exactly the mechanism predicted.

**§7.4 Compressed — works + one honest failure.** Distinct slopes cut **up to 99.94%**; last-level space **up to 81.2%**. Full compression: **≤52.2% space saved, ≤24.5% time lost** (the knob). **Longitude ε≥2⁹: compression BACKFIRES** (pointer overhead `m⌈log t⌉` > `64t` savings) → "turn it off." A negative result, reported plainly — matches the §4 reasoning.

**§7.5 Multicriteria — auto-tuner demonstrated.** `tol` = a **stopping criterion**, not a tuned param. Impl: power-law `aε^{−b}` (Levenberg–Marquardt), Newton root-finding, range `[8, n/2]`. Demos: Web logs → 1 MiB L2 → **ε=393, 10 iters, 19s**; IoT <500ns → **ε=432, 74.55 KiB, 6s**. **Everything <20s**; power-law fit ≤4.8% MSE.

**§8 Conclusions + future work (author-confirmed open seams):**
- Summary: <3s to index 91 GiB; ≤75% smaller than FITing-Tree; orders over B+-tree/CSS/RMI.
- ⚑ **"§5 Distribution-Aware left as future work — implementation AND experimentation."** → **§5 was never built.** Pure theory.
- ⚑ **Closed-form `s(ε)`** (key→space mapping) open — would boost Multicriteria.
- Also open: **DBMS integration** (esp. external-memory), **SIMD** search/update.

### Cheat sheet — §7–8
```
space   : optimal segmentation = 20–75% fewer segments, FREE; m 2–5 orders < n
query   : PGM◦REC beats trees & RMI on BOTH axes; predictable latency (MAE 226 vs RMI 892)
dynamic : write-heavy WINS 13–71%; read-heavy LOSES (15% @0.9) = read amp; 611–3891× smaller
compress: 81% slope space saved; Longitude ε≥2⁹ BACKFIRES (turn off)
tune    : all budgets solved <20s
open    : §5 NEVER IMPLEMENTED · closed-form s(ε) · DBMS/EM · SIMD
```

---

## 🔍 Research-Angles Capstone

> *Honesty labels throughout: "paper" = a result/claim in the paper; "open (author)" = the authors list it as future work; "mine" = my extrapolation, not a paper claim. Several of the strongest angles are mine — built by composing the paper's separate pieces and stress-testing against the real bottleneck.*

**The method that produced these (worth keeping):** propose a composition, then *don't stop at "sounds good"* — ask "does it actually help, and on which axis?" Identifying the **true bottleneck** (e.g. breadth vs depth) is the insight, and it tells you what a real fix must do.

### Tier 1 — highest-leverage, near-empty ground

**A1. Dynamic + distribution-aware via hot/cold tiering (mine; partly open by author).**
*The gap:* PGM has a distribution-aware **static** index (§5) and a distribution-blind **dynamic** shelf (§3) and **never marries them**. Dynamic reads pay a `log n` **breadth** penalty (search every run) that §5's **depth** optimization cannot touch — a hot key lives in one run, but you still probe the others.
*Why it's wide open:* **§5 was never even implemented** (§8, author-confirmed). So even the static variant lacks code/experiments, and the dynamic combination is untouched.
*The naive version (weak):* build each run distribution-aware. Free synergy — the shelf already rebuilds runs on merge, a natural moment to re-estimate `pi` and re-fit, so **drift fixes itself**. But it only shaves depth; the `log n` breadth factor remains. *Defensible one-liner: "distribution-awareness helps depth, but the shelf's cost is breadth."*
*The strong version:* use the distribution to decide **where keys live** — a small, cache-resident **hot front tier** searched first, cold keys aging into big runs. A hot query resolves in the tiny tier and **skips the cold runs** → attacks breadth directly. Borrow **hot/cold separation + workload-aware compaction** from the LSM/RocksDB literature (PGM imports none of it).
*Empirical hook:* §7.3 already shows the read-heavy loss (15% @0.9) this would target.
*Sweet spot:* **mixed workloads** (heavy writes + skewed hot reads). Pure read-skew → static §5, no shelf; pure writes → skew irrelevant.

### Tier 2 — solid, well-motivated

**A2. Beat read amplification with in-place / incremental updates (mine → this is ALEX's territory).**
*The gap:* §3 rebuilds a **whole run** for a single-key change (granularity mismatch) and taxes **every** read by `log n`, forever — confirmed by §7.3's read-heavy loss. The rebuild also *ignores* the locality §2 worked to exploit.
*Two sub-questions:* (i) Can the optimal segmentation be updated **incrementally** under the ε-guarantee, instead of rebuilt? (hard, valuable) (ii) Can read amplification be **bounded** while keeping cheap writes?
*Status:* this is exactly what **ALEX** (in-place gapped arrays) and **LIPP** attack — so you're entering an active line, but with the objection pre-formed. **Read ALEX as the direct answer to this seam.**

**A3. Closed-form `s(ε)` / key→space mapping (open, author-confirmed).**
*The gap:* §6 has no closed form for segment count, only Lemma 2's loose bound + an empirical power law. §8 lists a real key→space mapping as future work, both for theory and to **boost Multicriteria** (better guesses → fewer iterations, tighter guarantees).
*Why it's hard/interesting:* it asks for a predictive relationship between data regularity and `m` — a genuinely open theoretical question the authors flag.

**A4. Predictable / de-amortized worst-case writes (mine).**
*The gap:* "O(log n) amortized" hides that one unlucky insert merges half the data — bad for latency-sensitive/real-time systems. The paper's de-amortization (≤3 copies, spread work) is textbook; a learned-index-specific, **tail-latency-aware** scheme is open.

### Tier 3 — applied / incremental seams

**A5. Online/adaptive distribution-awareness under drift (mine; §5's own caveat).** §5 needs `pi` known in advance and freezes them; workloads drift. Incremental re-estimation + re-fit is open (and dovetails with A1's free-rebuild synergy).

**A6. Adaptive / cost-aware compression (mine; §7.4's failure).** Slope compression **backfires** when `m⌈log t⌉ > 64t` (Longitude ε≥2⁹). A predictor that decides **per-dataset/per-level whether to compress** (or chooses an encoding) avoids the regression — small but concrete.

**A7. Spatially-varying ε + interaction with §5/§6 (mine).** §6 picks **one global ε** by budget; §5 varies **tolerance by frequency**. A **per-region ε** (different regularity in different key ranges) is never explored, nor its interaction with §5/§6.

**A8. Systems (open, author-confirmed).** DBMS integration (especially external-memory), and **SIMD** search/update implementations.

### Priority read for "where's the open road"
1. **A1** (hot/cold tiered dynamic distribution-awareness) — strongest: composes two separate paper pieces, §5 is unbuilt, has a concrete breadth-attacking mechanism and a literature to borrow.
2. **A2** (in-place updates) — highest-impact but contested; ALEX/LIPP are here. Use it as the lens for reading ALEX.
3. **A3** (closed-form `s(ε)`) — cleanest *theory* gap, author-confirmed.
4. Then A4–A8 as supporting threads.

### Bridge to ALEX
Carry **one sentence** in: *§3 rebuilds whole runs for single-key changes and taxes every read by `log n` (measured: 15% slower read-heavy at 0.9) — the update granularity is wrong, and read-heavy workloads pay forever.* ALEX is the response: **absorb inserts in place (gapped arrays)** so reads stay at a single structure (no `log n` factor), trading gap management + occasional node splits. Read ALEX as the answer to seam **A2**, with **A1** (distribution-aware placement) as the thread ALEX does *not* fully address — your own open road past it.

---

## Master cheat sheet (whole paper)

```
ATOM (§2)      : everything = rank(x); optimal PLA (convex hull, Lemma 1, O(n)); recursive (fully learned)
                 Lemma 2: ≥2ε keys/segment ⇒ m ≤ n/2ε; Theorem 1: Θ(m) space, O(log m+log ε), I/O-optimal
DYNAMIC (§3)   : shelf of static PGMs (1,2,4,…); insert=binary-counter carry+merge (O(log n) amort);
                 delete=tombstone; read=search every run (×log n) — Theorem 2; the LSM bargain
COMPRESS (§4)  : intercepts succinct (Prop 1); slopes share via interval-stabbing (Thm 3); lossless, no decompress
DIST-AWARE(§5) : tolerance yi=min(1/pi,ε); telescoping routing → O(H) avg time, O(m) space (Thm 4); never built
MULTICRIT (§6) : single monotone ε → 1-D tune; power-law m≈aε^{-b}; measure don't model for time
EXPERIMENTS(§7): 20–75% less space free; beats trees+RMI; dynamic read-heavy loss CONFIRMS skepticism
OPEN ROAD      : A1 hot/cold dynamic dist-aware (mine) · A2 in-place updates (→ALEX) · A3 closed-form s(ε) (author)
```