# FITing-Tree: A Data-aware Index Structure — Notes

*Galakatos, Markovitch, Binnig, Fonseca, Kraska (SIGMOD 2019)*

---

## The One Big Idea

**An index is a function mapping keys → positions; approximate that function with `ε`-bounded linear segments instead of storing every key.** FITing-Tree is the Kraska learned-index idea made **bounded, updatable, and predictable** — its signature move is an error bound `ε` *declared at construction time* (not measured afterward), which acts as a single tunable knob between lookup speed and index size.

The whole paper hangs on three things:

| Concept | What it is | Why it matters |
|---|---|---|
| **`ε` (error knob)** | Max distance any key's predicted position can be from its true one | One dial trading space ↔ speed; the paper's identity |
| **Segment** | A run of keys where every key is within `ε` of its interpolated line | Stores 2 numbers (start key + slope) instead of all the keys |
| **Cost model** | Converts a latency SLA *or* space budget into the right `ε` | Makes the abstract knob usable by a real DBA |

---

## Abstract

- **Motivation = memory cost, not speed.** Indexes can consume **55% of a DBMS's memory** (TPC-C); the goal is to shrink the index while preserving lookup speed.
- **Core idea:** a piecewise-linear learned index with a **bounded error `ε` specified at construction time** (vs. Kraska, where error is measured *after* training).
- **`ε` is one tunable knob** on the space↔speed tradeoff: small `ε` → faster lookups but more segments (larger); large `ε` → fewer segments (smaller) but slower search. This is the "FIT" pun.
- **Key practical contribution:** a **cost model** that converts a latency target (e.g. 500ns) *or* a storage budget (e.g. 100MB) into the right `ε`.
- **Claim:** matches full-index lookup speed while cutting storage by **orders of magnitude** ("same speed, far less space").
- **Thread to PGM:** the "`ε` up front" philosophy is the seed PGM later formalizes into *provably optimal* segment counts.

## 1. Introduction

- **Why compression isn't enough:** classic B+-tree tricks (prefix/suffix truncation, Huffman) shrink each *node*, but index size still grows **linearly with the number of distinct keys** — `O(n)` is unavoidable.
- **Worst case = timestamps / sensor data** (IoT, AVs): huge and *ever-growing* key sets → indexes grow forever. DBA's only memory lever was dropping the index entirely.
- **The key architectural move:** replace fixed-size **leaf pages** (which enumerate keys) with **piecewise linear functions** (which approximate position over a trend). A line describing a trend ≪ a list of keys → orders-of-magnitude space savings.
- **Why it works:** exploits **trends in the data** (e.g. Fig 1: IoT timestamps are near-linear in day/night/weekend stretches) — same insight as Kraska's CDF, expressed as explicit segments.
- **Explicitly a "learned index" [Kraska], but claims 3 things RMI doesn't:**
  1. **Bounded worst-case lookup** (guaranteed by `ε` at construction — the headline differentiator).
  2. **Efficient inserts** (RMI was read-only).
  3. **Paging** — data need not be in one contiguous region (addresses a Kraska future-work gap).
- **Novelty is the *framing*:** piecewise approximation is old, but nobody applied it to *indexing operations* (lookups/inserts/ranges).
- **Orthogonal to node compression:** inner nodes are still a tree, so prefix/suffix truncation can stack on top.
- **Threads 1 & 2 (bounded error, inserts) are exactly what PGM and ALEX later build on.**

## 2. Overview

- **Index = monotonic function** mapping key → storage location. True function is too complex to learn exactly → **approximate it with disjoint linear segments**, bounded by error `ε`.
- **The compression trick:** each segment stores only **(1) starting key + (2) slope**. Find a position via **linear interpolation**: `pred ≈ start_pos + slope·(key − start_key)`. Two numbers summarize a whole stretch → orders-of-magnitude savings, **agnostic to key density** (stores the *trend*, not the keys).
- **Why linear (not polynomial):** far cheaper to compute → fast construction + fast inserts. Deliberate tradeoff (opposite of Kraska's NN-at-top).
- **Segment (formal):** a contiguous run of the sorted array where *every* key is within `ε` of its interpolated position. Fig 4: segment invalid if any middle point strays > `ε` from the line.

### Clustered index structure (Fig 2) — the key picture
- **Traditional B+-tree:** fixed-size leaf pages enumerate keys. **FITing-Tree:** **variable-sized segments**, each sized to satisfy `ε`; segments can be allocated **non-contiguously** (→ enables paging).
- **Variable sizing = adapts to data shape:** long smooth trend → one big segment; jumpy region → many small ones (vs. B+-tree's fixed grid).
- **Storage:** leaf nodes hold per-segment **(slope, start key, pointer)**; **inner nodes are identical to a normal B+-tree**.
- **Lookup = 2 phases:** (1) ordinary B+-tree descent to find the segment [exact], then (2) interpolate at the leaf + **local search within `ε`-window** [the learned part].
- **Modular:** inner structure needn't be a B+-tree — can swap in FAST for read-only (tested §7.4).

### Non-clustered / secondary index (Fig 3)
- Secondary attribute isn't sorted + has duplicates → add an **indirection layer ("Key Pages")**: pointer array, same size as data, sorted by indexed key. Segmentation runs over this.
- Lookup returns a position *in the indirection layer* → follow pointer to record (a **two-hop** lookup: predicted index-position → pointer → actual record).
- Indirection overhead is **not FITing-Tree-specific** (any secondary index needs it); still far smaller than a B+-tree (fewer leaf/inner nodes). The win is on the index over the layer, not the layer itself (the layer is `O(n)` for both).

## 3. Segmentation (ShrinkingCone)

### 3.1 Objective
- Reject least-squares (E2 / average error) — it doesn't bound the *worst* miss.
- Target **max error (E∞)**: guarantee *no* key is > `ε` off. `ε` = the worst-case search window, so it must be a hard bound. (vs. Kraska's measure-after-the-fact.)
- **Greedy, not optimal, by choice:** optimal segmentation is O(n³)/O(n). Need a fast **one-pass O(n), constant-memory** algorithm for cheap construction + inserts.

### 3.3 The ShrinkingCone algorithm
- Goal: grow each segment as **long as possible** while keeping all keys within `ε`.
- Maintain a **cone** of feasible slopes from a fixed **origin** (first key): bounded by a **high slope** and **low slope** = the family of lines still valid for the segment.
- **Per new key:** compute slope to its position **+ε** (steepest) and **−ε** (shallowest); update high = min(old high, new), low = max(old low, new). Cone only **narrows or holds** — never widens.
- **Exit:** key falls **outside** the cone → no line satisfies it + all prior keys → close segment, make this key the new origin (reset cone).
- Only tracks **2 numbers** (high/low slope) → constant memory, single pass, no backtracking.
- **Key invariant:** the cone summarizes *all prior constraints at once* — inside cone ⇒ within `ε` of a valid line; outside ⇒ some earlier key would break. No need to store past keys.
- Fig 5: pt1 origin → pt2 sets both slopes → pt3 inside, tightens upper only → pt4 outside, new segment.

### 3.2 / 3.4 Guarantees
- **Theorem 3.1:** a *maximal* segment covers **≥ `ε + 1` locations** (can't be too short).
- ⇒ at most ~`|D| / (ε + 1)` segments ⇒ **worst case, FITing-Tree ≤ a B+-tree with `ε`-sized pages.** Graceful degradation, never worse than B+-tree (the safety net; cf. Kraska's hybrid B-Tree fallback).
- **Caveat:** on *adversarial* data, ShrinkingCone can be arbitrarily worse than optimal in segment count — but still capped by the `ε+1` bound (never worse than B+-tree).
- **In practice (Table 1):** vs. optimal, ratios ~**1.05–1.6×** on real data (Taxi, OSM, Weblogs, IoT). Fast + bounded + near-optimal.

## 4. Index Lookups

### 4.1 Point queries — 2 steps
- **Step 1 — Tree Search (exact):** ordinary B+-tree root→leaf traversal; leaves are keyed by each segment's **first key**, value = **(slope, pointer)**. Cost = **O(log_b p)** where `p` = number of *segments*, NOT keys. Tree is short because segments ≪ keys → big space/speed win.
- **Step 2 — Segment Search (approximate):** interpolate with `pred_pos = (k − s.start) × s.slope` — **one subtract + one multiply** (the entire "model"). Then binary-search the guaranteed window `[pred_pos − ε, pred_pos + ε]` (width `2ε`). Guaranteed to find the key because ShrinkingCone built the segment to satisfy `ε`.
- **Bounded cost = the headline:** segment search is **O(log₂ ε)** (ε constant). Total lookup = **O(log_b p + log₂ ε)** — both terms small/bounded. This is the concrete proof of the intro's "bounded worst-case lookup" claim (the thing RMI couldn't guarantee).
- **Search is swappable:** linear / binary / exponential depending on hardware + `ε` size (exponential good when prediction usually very close).
- **Index vs. data are separate:** the segment is a *recipe* to predict position, not the data itself. The keys all still physically exist; the slope narrows `n` → a `2ε` window, then you read the real data to finish. (In the paging/disk variant, this final read is a *bounded* fetch — `ε` caps how much I/O you do.)

### 4.2 Range queries
- Reduce to a **point lookup for one endpoint** (start of range), then **scan forward** until out of range — no extra tree traversals.
- **Scan cost depends on clustering:** clustered → **sequential** access (fast); non-clustered → scan the sorted indirection layer but follow pointers to scattered records → **random** access (slower).
- Random-access penalty is **inherent to any non-clustered index** (a secondary B+-tree pays it too), not FITing-Tree-specific. Range cost dominated by **selectivity**.

## 5. Index Inserts

### The core tension
- Inserts threaten the **`ε` guarantee**: adding a key shifts all later positions → can push existing keys outside their `ε` window. So an insert must preserve `ε` for *every* key, not just find a slot. This is *the* reason learned indexes struggle with writes (B+-trees don't have this constraint).

### 5.1 In-place strategy (baseline)
- Split total `error = e + ε`: **`e`** = segmentation/fitting error, **`ε`** = insert budget (slots a key may move). [Note: `ε` reused here = insert budget, not global bound.]
- Page sized `|s| + 2ε`: data in the **middle**, `ε` empty slots at each end. Insert = shift keys toward nearer gap, drop in. When full → re-run ShrinkingCone, split into new segments.
- **Why it's bad:** avg **|s|/2 keys moved per insert**. Segments can be huge (the whole point), so big-`error`/uniform data = worst inserts. Classic trap: *big segments help lookups/space but kill inserts.*

### 5.2 Delta strategy (the real contribution)
- Each segment gets a small **fixed-size sorted buffer**. New keys → buffer (cheap, no shifting). When buffer **full** → merge with segment data + **re-segment** (ShrinkingCone) into new valid segments. **Amortizes** re-segmentation across many cheap inserts.
- **= the delta/buffer pattern** (like LSM memtable / column-store delta merge). **This is the canonical write-absorption idea that ALEX, PGM, LIPP all build on.**
- **Correctness trick:** buffer breaks `ε`, so segment with error **`error − buff`** (tighter fit). Guarantees even buffered keys stay within original `error` → **lookups stay oblivious** to data-vs-buffer. Clean design: pay for writes by tightening read-side fit.
- **Cost:** normal insert = **O(log_b p) + O(buff)**; buffer-flush = extra **O(d)** (d = segment + buffer size). High write rate → could double-buffer (out of scope).

### Research thread
- This section is the seed of the entire updatable-learned-index line. The delta-buffer + amortized re-segmentation idea → ALEX's gapped arrays/buffers, PGM's dynamic merging. The "big segments vs. cheap inserts" tension is exactly what those papers attack.

## 6. Cost Model

- **Purpose:** make `ε` *settable* from a real requirement. A DBA optimizes one of two objectives: **lookup latency** or **space**. (6.1 = latency; 6.2 = storage budget, the mirror image.)

### 6.1 Latency guarantee — "meet my SLA at smallest size"
- **Latency model (Eq 5):** `latency(e) = c·log_b(S_e) + log₂(e) + log₂(buff)` — three additive terms, one per lookup phase (mirrors §4–5):
  - `c·log_b(S_e)` = **tree search** (`S_e` = #segments at error `e`; `c` = cache-miss penalty ~50ns — dominates, since tree traversal hits scattered memory).
  - `log₂(e)` = **segment search** (binary search over the `2ε` window).
  - `log₂(buff)` = **buffer search** (delta buffer from §5).
- **`S_e` is the linchpin:** larger `e` → fewer segments → shorter tree (faster) but wider `2ε` window. Encodes the whole space/speed tradeoff. Get it by (1) **measuring** (segment your data at several `e`) or (2) **approximating** (e.g. segment count ∝ 1/error, roughly linear). Data-dependent.
- **Simplifying assumption:** `c` treated as constant (caching actually makes it variable) → model gives a *good* `ε`, not provably optimal.
- **Selection rule (argmin):** pick `e ∈ E` (e.g. {10,100,1000}) that **minimizes SIZE** s.t. **latency(e) ≤ L_req**. → hand it an SLA, get back a concrete `ε` + smallest index that honors it. This is the "abstract knob → real requirement" inversion promised in the abstract/intro.

### 6.2 Space budget — "fit my memory, be as fast as possible"
- Mirror of 6.1: **size = hard constraint, latency = minimized.** Directly answers the intro's "DBA can't cap index memory" problem.
- **SIZE function (Eq 7):** `SIZE(e) = S_e·log_b(S_e)·16B  +  S_e·24B`
  - **Tree term** `S_e·log_b(S_e)·16B`: B+-tree over segments; 16B/entry (8B key + 8B ptr). *Pessimistic* bound (over-counts → safe for budgeting).
  - **Segment term** `S_e·24B`: per-segment metadata = **start key + slope + pointer = 24B**. The compression claim in bytes.
- **Why the space win is precise:** `SIZE` depends only on **`S_e` (segment count), not `n` (key count)**. One segment absorbs many keys → `S_e ≪ n` → orders-of-magnitude savings. (B+-tree scales with `n`; FITing-Tree with `S_e`.)
- **Selection rule:** `e = argmin LATENCY(e)` over `{ e ∈ E | SIZE(e) ≤ S_req }`. Tight budget → larger `e` (fewer/bigger segments); generous budget → smaller `e` for speed.

### Cost model — both halves together
| Section | DBA gives | Constraint | Minimizes |
|---|---|---|---|
| 6.1 | latency SLA `L_req` | LATENCY ≤ L_req | SIZE |
| 6.2 | space budget `S_req` | SIZE ≤ S_req | LATENCY |
- Both run off the **same engine: `S_e` (segments vs. error)**. Measure that one curve → both directions fall out. This is the "abstract knob → real requirement" inversion fully realized.

## 7. Evaluation

### Setup
- 3 baselines: **Full index** (dense, best-case speed, huge, fixed size) · **Fixed-size paging** (sparse B+-tree, tunable — the fair competitor) · **Binary search** (no index, size 0, the floor).
- FITing-Tree + fixed-paging are tunable → charts plot **latency vs. index size** (sweep the knob).

### The key concept: periodicity / non-linearity (Fig 9)
- Performance hinges on data **periodicity** = distance between "bumps" in the key→position function.
- `ε` **> periodicity** → bump fits → **one segment** (great). `ε` **< periodicity** → **many segments** (poor).
- **Non-linearity ratio:** segments needed vs. worst-case data. Datasets chosen to span it: **IoT** (one strong bump, day/night) · **Weblogs** (multiple: daily/weekly/yearly) · **Maps** (nearly linear, longitude). *More linear → FITing-Tree does better.*

### Exp 1 — Lookups (Fig 6) — HEADLINE
- FITing-Tree **always beats fixed-paging** (faster at any size).
- **Space savings (remember these):** Maps — matches full-index speed at **609MB vs 30GB+**; a **1MB** FIT matches a **10GB+** fixed-paging index = **4 orders of magnitude**.
- Curves: tiny index → degrades to binary search; large index → converges to full-index speed. Maps converges fastest (most linear).

### Exp 1c + 2 — Inserts (Figs 7, 8)
- Throughput **comparable to fixed-paging**; **full B+-tree faster** (no segmentation/merge) — honest cost of compression. At large error FIT can beat fixed-paging (fewer re-segmentations).
- **Delta vs in-place (confirms §5):** delta wins for **error > 100** (in-place shifts |s|/2 keys); in-place wins for **small error** (tiny segments). Low fill factor → best in-place throughput. → default to delta.

### Exp 3–5
- **Build (Fig 10):** ~**4.2s** constant overhead vs fixed-paging (segmentation scans all elements); *avoidable* via streaming load (can even be cheaper — fewer leaf entries).
- **Other inner indexes (Fig 11):** swap STX-tree → **FAST** (faster, bigger) or **lookup table** (smallest, slower). Confirms modularity.
- **Scalability (Fig 12) — THE THESIS:** at scale factor 32, **full + fixed-paging don't fit in RAM; FITing-Tree does.** Space savings = "fits vs doesn't." Callback to 55%-memory intro.

### Exp 6 — Worst case (Fig 13)
- Adversarial **step function (step=100)**. error < 100 → one segment/step → **same size as fixed-paging**; error > 100 → **single segment**, size plummets.
- Confirms Thm 3.1: **never more leaf entries than fixed-paging** → graceful degradation, never worse than B+-tree (the de-risking guarantee).

### Exp 7 — Cost model accuracy (Fig 14)
- `c = 50ns` (measured). **Latency model** = accurate **upper bound**, slightly over (ignores caching) → SLA always met. **Size model** accurate + **pessimistic** → budget never blown.
- **Both err in the safe direction** → trustworthy as an SLA tool, not just a heuristic.

## 8. Related Work

- **Recurring delta (FITing-Tree's identity by contrast):** prior work either (a) ignores the **data distribution**, (b) doesn't **bound error/latency**, or (c) doesn't support **index operations (lookup/insert)**.

### Index compression
- **B+-tree compression** (prefix/suffix truncation, dictionary, key normalization): shrinks keys *in nodes* → **orthogonal + stackable** with FITing-Tree (trends vs. encoding).
- **Correlation Maps:** need an existing primary index; use fixed buckets. FIT needs none + uses **variable-sized** segments.
- **FAST / ART / hot-cold / LSM:** hardware/workload-specialized trees — **orthogonal**, usable as FIT's *internal* structure (demoed w/ FAST §7.4).
- **BF-Tree (bloom leaves) + sparse indexes (Hippo, BRIN, SMAs):** also store *region* info (closest to segments) but **don't exploit distribution or bound latency.**

### vs. Kraska learned indexes [30] — KEY PARAGRAPH
- FITing-Tree's 3 explicit contributions over RMI: **(1) strict error guarantees** (bounded by construction vs. measured after) · **(2) insert support** (RMI read-only) · **(3) cost model** (predictable latency/size). → FIT = "Kraska made bounded, updatable, predictable."

### Partial / adaptive indexes (save space by indexing *less*)
- **Partial / Tail indexes:** index only a subset; FIT indexes all but *could* index only "important" ranges.
- **Database cracking:** reorders by *past* queries → bad at **ad-hoc**; FIT handles ad-hoc fine.

### Function approximation (the math lineage)
- Piecewise-linear approx is **old**; FIT's choices = **monotonic + E∞ (max error) + disjoint segments**. Delta: **never applied to indexing** (no lookup/insert).
- Time-series PLA: trades segments for accuracy but **no guarantees**.
- Inverted-list compression w/ linear fns: uses **linear regression (E2) → no error bound** (cleanest contrast: bounded E∞ vs. unbounded E2).

## 9. Conclusion
- FITing-Tree = tunable **error knob** (balance speed vs. space) + **cost model** (set knob from latency or storage requirement). Matches full-index speed at **orders-of-magnitude** less space on real data.

---

## 10. Research Angles & Opportunities

*Threads left loose by FITing-Tree, connected forward to where the field picked them up. These are the seams worth poking for an angle.*

### Loose threads in the paper itself
- **ShrinkingCone is greedy, not optimal — and can be *arbitrarily* worse than optimal on adversarial data.** The `ε+1` bound caps the absolute worst case (≤ B+-tree), but the gap-to-optimal is unbounded in theory. *PGM closes exactly this* by computing provably (near-)optimal segment counts. Open question still worth probing: optimal-vs-greedy tradeoffs under *updates*, not just static build.
- **The insert strategy is conservative.** Delta-buffering + full re-segmentation on flush is a first stab; the "big segments help reads but kill inserts" tension is unresolved here. *ALEX (gapped arrays) and PGM (logarithmic merging) attack this directly* — and the recent robustness benchmarks (Wongkham 2022, Luo 2025) show it's *still* not fully solved under concurrency/adversarial workloads.
- **Cost model assumes a constant cache-miss penalty `c`.** Real `c` varies with cache behavior and access pattern; the model deliberately over-estimates to stay safe. A more accurate, cache-aware cost model (or a learned `c`) is an open refinement — and matters more on disk/SSD where the "miss" cost is a real I/O.
- **Paging is sketched, not built.** Non-contiguous segments *enable* disk-resident data, but the paper doesn't actually evaluate it. **This is the seam I flagged while reading:** what happens when the final local-search read is a disk fetch? `ε` bounds the over-read, but mispredictions cost real I/O. *This is precisely Lane 3 of the reading list* (BOURBON, the disk-resident eval, DobLIX).

### Connecting forward to your reading list
- **→ PGM (read next):** takes FITing-Tree's `ε`-bounded-segments idea and makes the segment count *provably optimal* + the structure *fully dynamic*. Reading FIT first means PGM's piecewise-linear core will already be familiar — you'll see it as "FIT with optimal segmentation + a recursive multi-level structure + proper update support."
- **→ ALEX:** picks up the *insert* thread. Where FIT re-segments on buffer flush, ALEX uses gapped arrays + model-based inserts to avoid the wholesale re-segmentation cost. The "big-segment insert penalty" is the exact problem ALEX is designed around.
- **→ Disk/LSM lane:** the unbuilt paging promise is the open frontier. FIT proves `ε` bounds the search window; making that translate to *bounded, cheap I/O on SSD* (not just bounded memory reads) is what BOURBON/DobLIX/the disk-eval papers work on.
- **→ Security lane (Yang 2024):** FIT's worst-case guarantee (`≤ B+-tree`) is *static* — proven over a fixed dataset. The adversarial-insert attacks on ALEX show that *dynamic* worst-case guarantees are much harder. A FIT-style construction-time bound that *also* holds under adversarial insertion is an open design target.

### Candidate angle (specific)
- **"Construction-time `ε`-bounds that survive updates and disk."** FIT gives a clean static guarantee; PGM gives optimal static segments; neither fully delivers a *bounded-worst-case, update-resilient, disk-aware* learned index. The intersection — a piecewise-linear index with FIT's predictable cost model, PGM's optimality, ALEX's cheap inserts, *and* an SSD-aware cost term — is a concrete, tractable target. Read PGM + ALEX + the disk-eval paper together and look for what none of the three simultaneously guarantees.