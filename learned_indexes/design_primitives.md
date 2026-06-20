# Learned-Index Design Primitives — my parts bin

*A living note. Every learned index = a set of reusable design choices. This catalogs the "parts" across the papers I've read (RMI, FITing-Tree, PGM, ALEX) so I can recombine them. Update the **stance** column as I go: ✅ like · ❌ dislike · 🤔 undecided · ⭐ want to use.*

---

## What I've flagged so far

| Primitive | Source | My stance |
|---|---|---|
| Shelving (shelf of static indexes, merge on overflow) | PGM §3 | ✅ |
| Hot/cold tiering (temperature-based placement) | mine (A1) | ✅⭐ |
| Fully-learned recursive hierarchy (no B+-tree) | PGM §2.2 | ✅ |
| Per-node splitting (nodes adapt independently) | ALEX | ✅ |
| Hard ε error bound (guaranteed worst case) | PGM / FIT | ✅ |
| No error bound (emergent error) | ALEX | ❌ |

---

## The design axes (every index picks one cell per row — mark mine)

This is the whole design space on a page. My "design" = a path through it; **empty/unusual cells = open ground.**

| Axis | Options (→ harder/newer) | My pick |
|---|---|---|
| **Error** | stored-after-training (RMI) · **hard ε bound** (PGM/FIT) · emergent + exp-search (ALEX) | hard ε ✅ |
| **Last-mile search** | fixed 2ε binary (PGM) · exponential, error-adaptive (ALEX) | ? — *exp-search is nice even WITH a bound* |
| **Structure** | learned + tree fallback (RMI/FIT) · **fully learned recursive** (PGM) | fully learned ✅ |
| **Updates** | rebuild-shelf (PGM) · gapped array in-place (ALEX) · delta buffer (FIT) | ? — shelf ✅ but see synthesis |
| **Adaptation** | static/none (PGM static) · reactive cost-model (ALEX) · proactive/predictive (open) | ? — proactive ⭐? |
| **Placement** | value-sorted (all) · recency runs (PGM shelf) · **temperature/hot-cold** (mine) | hot/cold ✅ |
| **Tuning** | manual (RMI #models, PGM ε) · auto from budget (PGM §6) · RL (LITune) | auto/RL ⭐? |
| **Granularity** | whole-index rebuild (PGM run) · **per-node** (ALEX) | per-node ✅ |
| **Record storage** | in-leaf / clustered (ALEX, B+-tree) · indirected key+pointer / unclustered (secondary index, heap) | ? — *flips with payload size; see note* |

---

## Full primitive catalog (by concern)

### A. Model-fitting & error
- **Optimal convex-hull segmentation** — PGM §2.1. Min #segments, *floating* line, ε guaranteed, O(n). *Benefit:* smallest + provable. *Cost:* batch (rebuild to update). **✅ (pairs with shelf)**
- **Greedy ShrinkingCone** — FIT. *Pinned* line, near-optimal, dead simple. *Cost:* more segments. 🤔
- **Per-node least-squares model** — ALEX/RMI. *Benefit:* trivial, adaptive. *Cost:* no bound. ❌ (the bound part)
- **Per-key error budget `yi=min(1/pi,ε)`** — PGM §5. Tolerance ∝ 1/frequency → hot keys tight/fast. ⭐ *This is the per-key cousin of my hot/cold idea — strongly consider.*

### B. Last-mile / search
- **Fixed 2ε binary search** — PGM. Safe, bounded, simple. ✅
- **Exponential search** — ALEX. Cost = O(log d) in *actual* error d → near-free when models are good. 🤔 *Note: exp-search composes with a hard bound too — exp-search INSIDE a guaranteed window = adaptive AND safe. Keep this.*

### C. Structure & routing
- **Fully-learned recursive hierarchy** — PGM §2.2. Segments index segments; navigation *computed* (eval line), no pointers/separators. *Benefit:* tiny, variable fan-out. ✅
- **Computed vs stored navigation** — the deeper primitive: route by *evaluating a model*, not following a pointer. ✅
- **Learned + tree fallback** — RMI (B-tree), FIT (B+-tree on top). Hybrid; simpler to reason about, bigger. ❌ (I prefer fully learned)

### D. Updates
- **Bentley–Saxe shelf** — PGM §3. Shelf of static indexes, binary-counter merge. *Benefit:* reuses optimal static build, cheap amortized writes. *Cost:* read ×log n (breadth). ✅ *but the breadth tax is the thing to fix.*
- **Gapped array** — ALEX. Scattered empty slots absorb inserts in place. *Benefit:* no rebuild, single structure (no log n). *Cost:* gaps fill → degradation; no bound. 🤔 *Like the in-place idea, want it WITH a bound.*
- **Model-based insertion** — ALEX. Place data where the model predicts (data conforms to model). *Benefit:* keeps predictions self-consistent. *Cost:* degrades under drift/crowding. 🤔 *double-edged — the seam I found.*
- **Delta buffer per node** — FIT. Small sorted buffer, flush+retrain. *Cost:* slow for big models, skew thrashes buffers. 🤔
- **Tombstone deletes** — PGM. Delete = disguised insert; free GC at merge. *Benefit:* uniform append-and-merge. ✅ *clean, reusable.*
- **Append-only fast path** — PGM §3. Sorted inserts just extend the last segment, O(1) amortized. ✅ *cheap special case worth keeping.*

### E. Adaptation & tuning (the *policy* layer — separable from structure)
- **Cost-model-triggered repair** — ALEX. *Measure* per-node cost, split/expand/retrain *selectively*, workload-aware. *Benefit:* spends effort where it hurts. *Cost:* reactive → repairs after paying. ✅ *like the mechanism; want it proactive.*
- **Multicriteria auto-tuning of ε** — PGM §6. Pick ε from a space/time budget (power-law + root-find). ⭐
- **Distribution-aware telescoping routing** — PGM §5. Weight first-keys by `qa,b/Pa,b` so hot keys are fast at *every* level, not just the leaf. ⭐ *directly relevant to hot/cold.*
- **Meta-primitive: separate mechanism (structure) from policy (when to adapt).** ALEX bolts a policy layer on; PGM's policies are fixed. A swappable policy layer is itself a design choice. ⭐

### F. Placement & organization
- **Value-sorted layout** — all. Default.
- **Recency-based partitioning** — PGM shelf. Runs by insert time (distribution-blind). 🤔 *the thing hot/cold replaces.*
- **Temperature / hot-cold tiering** — mine (A1). Hot keys in a small front tier searched first → attacks *breadth*. ✅⭐

### F2. Record storage & layout (where the actual records live)
- **In-leaf / clustered** — ALEX data nodes, clustered B+-tree. Record sits at the search position. *Benefit:* one access on lookup; **sequential range scans** (records physically adjacent). *Cost:* inserts **shift records** (big entries → expensive, lower fan-out). 🤔 *This choice is WHY ALEX needs gapped arrays + model-based insertion — both exist to make in-leaf inserts survivable.*
- **Indirected (key + pointer) / unclustered** — secondary index over a heap. Leaf holds (key, pointer); record stored elsewhere. *Benefit:* cheap inserts (append record, insert tiny pointer); small leaves → high fan-out → shallow tree; records sharable by many indexes. *Cost:* **pointer chase** per read (random access); range scans scatter → random access *per result*. 🤔
- **Deciding variable = payload size.** Small payload (≈ pointer size, e.g. ALEX's 8-byte values) → in-leaf wins (locality, ~no size penalty). Large payload (rows/blobs) → indirection wins (keeps nodes small/cacheable). *(B-tree vs B+-tree is the micro-version: B+-tree = keys-only internal nodes, all records in leaves → high fan-out, scan-friendly. ALEX is B+-tree-shaped: model nodes route, data nodes hold records.)*
- **Ordered leaf-level scan (range locality)** — the B+-tree property I like: all data at one level, walk it in order. Two ways: **array-backed** (PGM — data is one sorted array, scan = increment a pointer, NO inter-leaf pointers, cache-best, but hard to update in place) vs **linked-node** (B+-tree/ALEX — sibling pointers between leaf nodes, updates easier, one pointer hop per boundary). *Who has it:* PGM **best** (contiguous), ALEX good-but-gap-taxed (loses past ~1000-key scans), B+-tree great, **LIPP poor** (data scattered across ALL levels → must tree-walk; the unified-node price). ✅⭐ *Want PGM-class scans → keep data dense; gapping the data reintroduces ALEX's tax.*

**⭐ Recombination notes (ties to my seams):**
- *Indirected layout defuses part of the model-based-insertion degradation seam:* "place at the predicted position" means moving a **pointer**, not a record → cheap even under heavy/drifting inserts. Price: a pointer chase on every read. So *indirected + model-based placement* = trade read locality for cheap, drift-resistant placement — a different point on the curve than ALEX picked.
- *It makes "local split with ε" cheaper:* splitting a node of **pointers** is far lighter than splitting a node of **records** (no record re-placement). If I want cheap local splits AND an ε bound, an indirected layout may make the "local" part cheap — at a scan-locality cost to weigh.

### G. Guarantees (the things hardware can't buy)
- **I/O-optimality via lower bound** — PGM. Provable, not "fast in tests." ✅
- **Entropy-optimal average time** — PGM §5. ⭐

---

## ⭐ What my picks already sketch (the synthesis)

My likes line up *too* well to be coincidence. Put them together:

> **A fully-learned, error-bounded, hot/cold-tiered dynamic index whose nodes split locally while preserving ε.**

That's PGM's guarantees + PGM's fully-learned hierarchy + ALEX's per-node local adaptation + my hot/cold tiering — *minus* ALEX's missing bound.

**The central tension this exposes (= my real research question):**
- ALEX can **split a node locally** but has **no ε guarantee.**
- PGM **guarantees ε** but updates by **rebuilding a whole run** (no local split).
- **Nobody does both.** → *Can a node split/expand locally while preserving a hard ε bound?* If yes, you'd get ALEX's cheap local updates AND PGM's worst-case guarantee — and drop the read-amplification of the shelf.

This ties my earlier seams together: **A2** (in-place updates without losing the guarantee) + **A1** (hot/cold placement to kill breadth) + the ε bound I refuse to give up. The throughline of everything I like is: *local, bounded, distribution-aware adaptation.*

---

## To add as I read on
- **RadixSpline** — single-pass spline build, radix table routing. (primitive: spline fit; one-pass construction)
- **LIPP** — precise-position models (zero last-mile search via exact placement + collision chaining). (primitive: exact-placement models — contrast model-based insertion)
- **BOURBON** — learned models inside LSM levels. (primitive: where to put learning in an existing engine; A1 ground)
- **Morphtree** — nodes that flip read-/write-optimized layout per workload. (primitive: polymorphic node layout — close to my tiering)
- *Each new paper: add its primitives here, mark the stance, and check whether it resolves the local-split-with-ε tension above.*