# ALEX Notes — complete (§1–8)

*Ding et al., SIGMOD 2020. The updatable, in-place learned index. Read against PGM.*

> **Figures** (in outputs):
> - `alex-architecture.svg` — adaptive RMI, pointer routing, variable resolution (read first).
> - `alex-data-node.svg` — gapped array + model-based insertion + exponential search (the core trio).
> - `alex-repairs.svg` — expand vs split-sideways vs split-down.
> - `alex-cost-model-flow.svg` — the insert decision flow (the reactive policy + my thesis critique).
> - `alex-vs-pgm-thesis.svg` — the contrast and where my thesis sits.

---

## One big idea
RMI + **gapped arrays** + **model-based insertion** + **exponential search** + a **cost model** that reshapes the tree on the fly → a fully dynamic learned index. **No ε bound** — error is *emergent*, kept small reactively. ALEX = "take Kraska's read-only RMI and make it survive writes by co-designing storage, search, and an adaptive structure."

## The two weaknesses it fixes (in Kraska)
1. **Expensive inserts** — Kraska's one dense array shifts on every insert (O(n)).
2. **Decaying predictions** — models fit to original data drift as new data arrives.
Every ALEX mechanism attacks one of these.

---

## Node layout (§3.2)
- **Data nodes** (leaves): linear model (2 doubles) + gapped **keys** array + parallel gapped **payloads** array + occupancy **bitmap** (skip gaps on scan). Records **in-leaf / clustered**. Payloads can be pointers for large records (→ unclustered, at scan-locality cost).
- **Internal nodes**: linear model + **stored pointer array**; model **computes the index** into that array (no key comparisons), then dereference. **Power-of-2** pointer count; **multiple pointers may alias one child** (room to split later). Internal models are **exact by construction → no search in internal nodes**.
- vs PGM: PGM routes into a *contiguous* array by arithmetic (no pointers); ALEX computes an index but into a *pointer array* → children can be heterogeneous/movable. **Pointers are the price of in-place mutability.**

## The leaf trio (§3.1, the heart)
- **Gapped array**: gaps scattered *between* elements (B+-tree puts free space only at the end). Absorbs inserts locally + lets records sit at predicted positions.
- **Model-based insertion**: place each key WHERE THE MODEL PREDICTS (data conforms to model) → predictions stay self-consistent → error small. *Double-edged: gaps fill / drift grows → error climbs (the degradation seam).*
- **Exponential search**: probe outward in doubling jumps from the prediction, then binary-search the bracket. Cost **O(log d)** in the *actual* error d → near-free when models are good; also removes stored error bounds. *No ε guarantee* — adapts to error rather than capping it.

---

## Operations (§4)
- **Lookup**: descend (1 pointer chase/level, internal models exact → no search) → data node → exp-search.
- **Insert (non-full)**: model-predict pos → exp-search if off → drop in gap, else shift to nearest gap. O(log n) whp.
- **Insert (full)** = density band `dₗ=0.6, dᵤ=0.8`. "Full" = next insert exceeds `dᵤ`.
  - Expected vs empirical cost **< 50%** deviation → **expand + rescale** (no retrain) — the common case.
  - Deviates **> 50%** → cheapest of {expand+retrain, split sideways, split down}. Always splits in **2**.
- **Cost model** (§4.3.4): stats = (a) avg exp-search iters, (b) avg shifts/insert. lookup ∝ a; insert ∝ a+b. Plus a **TraverseToLeaf** model (depth + total inner-node size → pointer-chase + cache cost). All linear → decisions are **additive** (→ local).
- **Delete** (§4.4): lookup + remove, **no shifts → no model degradation** (asymmetry with inserts). Contract at `dₗ`. **Merge described but NOT implemented** → never reclaims depth.
- **Out-of-bounds / append-only** (§4.5): expand/new root; append detector expands right *without* model-based re-insertion.
- **Bulk load** (§4.6): greedy top-down; the **Fanout Tree** picks each node's power-of-2 fanout (grow FT levels until cost rises, then local merge/split of FT nodes to a min-cost cover). *Local FT merge/split mirrors my split/merge idea — but at build time, cost-driven.*

## Split types (the machinery I'm lifting) — `alex-repairs.svg`
- **Expand**: same node, bigger gapped array, model rescaled. Parent untouched, depth unchanged. *Density fix.* O(m).
- **Split sideways**: → two siblings; repoint parent (split redundant pointers, else double the array). Can cascade to root. Worst case **O(m·⌈log_m p⌉)**.
- **Split down**: node → internal + 2 child data nodes. **Adds resolution locally, no parent involvement** (B+-tree has no analog). O(m). ← closest to my local-split idea.
- **Power-of-2** = "boundary-preserving" split → **no retrain below**.

---

## §5 Analysis (read in depth)
- **Theorem 5.1 (RMI depth bound)**: depth ≤ **⌈log_m p⌉**, where p = densest-subregion partition count. Bounded by **density of the densest subregion**, NOT by #keys (contrast B+-tree). Maintained under inserts via the policy: expand → split sideways (hold depth) → split down only when forced.
  - **Authors hedge it**: "does not reflect worst-case guarantees in practice." ← ALEX's only bound, and they admit it's a proxy. *This is my opening.*
- **Complexity (§5.2)**: TraverseToLeaf ⌈log_m p⌉; lookup exp-search O(log m) worst / O(1) best; insert non-full O(log m) whp / O(m) worst; expand O(m); split-down O(m); **split-sideways cascade O(m·⌈log_m p⌉)** ← the worst-case insert cost my design must match/beat.

## §6 Evaluation (skim, grab caveats)
- Beats B+Tree across the read-write spectrum; beats Kraska on read-only by up to 2.2× (15× smaller); huge index-size wins (often 100s–1000s×).
- **Honest caveats worth keeping**:
  - **Range queries**: ALEX loses to a *re-tuned* B+Tree once scan length **> ~1000 keys** — gapped-array overhead outpaces its large-node scan-locality advantage. *(I use gapped arrays too → inherit this.)*
  - **Table 3 (actions when full)**: overwhelmingly **expand+scale** (e.g. longitudes 26157 vs 79 sideways vs 0 down). Splits are rare *when there's no radical shift* — supports ALEX's common case AND exposes its dependence on "no radical shift."
  - **§6.2.6 distribution shift**: claims robustness (3.2× on shift, 3.6× on sorted inserts) — but tested ONE pattern (init smallest 50M, insert rest) on *shuffled* data. The 2022–25 robustness papers stress it harder and find it cracks. *Know exactly what their eval did/didn't cover.*
  - Tail latency tunable via max node size (16MB → ~2µs p99).

## §7 Related work (don't skip the tree-index ancestors)
- **CSS-tree**: pointerless navigation via **arithmetic** (PGM's computed routing, *pre-learned-index*).
- **CSB+-tree**: extends CSS to support **incremental updates** without losing cache performance.
- → "pointer-free + updatable" has **non-learned ancestors**. My novelty must be framed as *pointer-free + updatable + ε-guaranteed + learned* — cite these so it doesn't look like reinvention.
- FITing-tree / BF-tree replace B+-tree leaves with models / bloom filters.

## §8 Conclusion + future work
- Authors list **"open theoretical problems about ALEX performance"** and **concurrency** and **larger-than-memory** as future work → they *admit* the theory is incomplete (my opening) and punt concurrency (my honest limit).

---

## 🔍 Research angles & skepticism
1. **The cost model is the fragile heart.** Magic constants (`dₗ=0.6, dᵤ=0.8`, 50% threshold), justified only as "works in our experience." Reactive + approximate; §4.3.6 admits it mis-predicts *exactly* under distribution shift. The robustness reckoning (2022–25) is the field confirming this in print. → *Replace heuristic triggers with a hard invariant.*
2. **No ε bound = no worst-case robustness.** Error is emergent; an adversary can drift a node just under the trigger and degrade quietly. PGM can't degrade past ε. → the core of my thesis.
3. **Merge unimplemented.** ALEX never reclaims depth after deletes → asymmetric, and exactly my Hard Part 2 (merge-on-underflow). *Nobody did this — not even ALEX.*
4. **Range-query cliff (>1000-key scans).** Gapped arrays hurt long scans. Open: gap layouts / hybrid leaves that keep short-scan locality without the long-scan penalty.
5. **Concurrency** is punted (§8) — XIndex/FINEdex territory; a separate hard axis.
6. **Append-only handling is ad-hoc** (detector + expand-right). PGM's clean append fast-path is more elegant — room for a principled treatment.

## 🎯 Thesis hooks (going forward) — see `candidate-thesis-local-split.md`
- **Trigger**: ALEX's cost-model + 50%-deviation is reactive/heuristic → my **ε-invariant** replaces it (deterministic, no constants). *"Trade a cost-model policy for a hard invariant."*
- **Split boundary**: ALEX splits **in half by key space** (a guess) → mine cuts at the **ε-forced boundary** (convex hull says where); sub-range monotonicity proves both halves ε-valid (free lemma).
- **Split-down** is the closest existing "add resolution locally, no parent disturbance" move — adapt it, but pointer-free (gapped SEGMENT array, since I have no pointer array to halve).
- **Merge-on-underflow** = my Hard Part 2; ALEX punted it → clean "nobody did this."
- **CSS/CSB+-tree** = cite as pointer-free + updatable ancestors; my novelty = + ε-guaranteed + learned.
- **Range-query cliff** = I inherit gapped-array scan overhead → must measure/own it.
- **Latency vs bound**: ε bounds positions not time → keep ε as the invariant I *prove*, latency as the *evidence* I measure.

---

## Cheat sheet
```
ALEX = adaptive RMI + gapped arrays + model-based insert + exp-search + cost model. No ε bound.
internal: model computes index into POINTER ARRAY (power-of-2, multi-pointer). exact → no search.
data node: model + gapped keys + gapped payloads + bitmap. records IN-LEAF. exp-search O(log d).
insert non-full: predict → gap? drop : shift to nearest gap.  O(log n) whp.
insert full (>dᵤ=0.8): <50% dev → expand+scale ; else cheapest{expand+retrain, split sideways, split down}.
split sideways = two siblings, repoint parent, cascade O(m·log_m p). split down = +1 depth locally.
delete: no shifts → no degradation. MERGE unimplemented. depth bound ⌈log_m p⌉ (Thm 5.1, hedged).
eval: beats B+Tree/Kraska; LOSES range >1000-key scans; Table 3 mostly expand; shift-robustness lightly tested.
ancestors: CSS-tree (pointer-free), CSB+-tree (pointer-free + updatable). FITing/BF-tree (model/bloom leaves).
PGM = computed+contiguous+pointer-free+ε-guaranteed+rebuild-to-update
ALEX = pointer array+in-place gapped+exp-search+cost-model-reshaped+no bound
mine = computed+contiguous (gapped SEGMENT array) + ε-guaranteed + local in-place ε-split
```