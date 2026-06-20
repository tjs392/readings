# Research Compass — learned indexes (a stance, NOT a commitment)

> **Purpose.** A *living research stance*, not a locked thesis. Records what I want and *why*, what to borrow, what to avoid, and the direction I'm circling — refined as I finish the literature. **I won't commit until the "still need to read" list is done.** Judgment without premature deciding.
>
> **How this doc is organized:** §1–4 = my values (wants / borrow / avoid / the gap). §5 = the core candidate. §6 = **Ideas Ledger** (every idea I've had, tagged ⭐ promising / 🔧 optional / 🗄️ set-aside). §7 = **Design Razors** (the principles I keep rediscovering). §8 = reading queue.

---

## 1. What I want (desiderata) + why

Each is a *goal with a reason*, not a design. **⚠️ They trade off — I can't maximize all.** The Tradeoff column is the honest part. Deepest recurring tensions: **slack/gaps (cheap in-place updates) vs density (space + fast scans)**, and **rigid/simple (concurrency + predictable writes) vs adaptive/clever (performance + robustness)**.

| I want… | Why (and who showed me) | Tradeoff — what it restricts |
|---|---|---|
| **Hard worst-case guarantee** (ε-bound) | Predictability + robustness under drift. The 2022–25 robustness reckoning says this is *the* missing thing. PGM has it; ALEX/LIPP don't. **Non-negotiable.** | Bounds *positions*, not latency. Pay for it on *every* key. Tighter ε → more segments → more space. A fixed bound can't *adapt* to workload. |
| **In-place updates / no read amplification** | PGM's shelf taxes *every read* `log n` — why it loses to ALEX on reads. Updates shouldn't scatter data. | Forces **pointers** (lose cache) **or gapped arrays** (lose space + scan tax) **or rebuild** (the shelf I avoid). Also makes **concurrency harder**. |
| **Pointer-free / computed navigation** | Cache wins latency (`log²N`-beats-`logN`: big-O hides the ~80ns miss). PGM's arithmetic routing > ALEX's pointer chase. | Requires **contiguous, uniform** layout → **fights in-place mutation**. *The* central tension → forces gaps or rebuilds. |
| **Predictable / bounded-cost writes** (no tail spikes) | LIPP's bulk adjustment + ALEX's big-node splits spike to tens of µs. Amortized-but-spiky hurts tails *and* concurrency. (A4.) | De-amortizing = **steady overhead vs rare big work** → can cost *average* throughput. *Local* merge may reclaim optimality less effectively → weaker `c·m`. |
| **Distribution-agnostic** (no global shape) | LIPP's global `G` is a static single-shape bet that goes stale under drift. Piecewise handles arbitrary/shifting CDFs structurally. | Piecewise = **more segments = more space** than one well-fit global model. Give up "one perfect model" compactness for robustness. |
| **Workload-adaptive** (hot/cold) | Real access is Zipfian. Put hot keys where they're cheap to reach. (A1.) | Adaptivity = **stat-tracking + restructuring** → fights **predictable writes AND concurrency**. Adds knobs (the thing I criticize in ALEX). |
| **Concurrency-approachable** (even if v1 defers it) | Don't paint into LIPP's corner. Bounded-local writes leave room for optimistic-read + slot-locks later. | Wants **small, bounded, simple** writes → fights **rich adaptivity**. Pushes toward more *rigid* structures. |
| **Ordered leaf-level scan** (range locality) | B+-tree links leaves → range scan walks one level. PGM gets it best (contiguous array). | Requires data **dense at one level** → **fights gaps** (ALEX's >1000-key scan cliff) and **fights in-place**. Best scans ↔ hardest updates. |
| **Space-honest / small** | LIPP's lookup win is partly a *space* trade (equal-memory ALEX-M beats it). Any win must hold at *equal memory*. | Small space **fights gaps** (needed for in-place), **fights hot/cold redundancy**, fights insert-slack. (The ALEX-M fill-factor dial.) |

**Meta-point:** "in-place + pointer-free + dense + good-scans" is *over-constrained*. My design problem is *choosing where on these tradeoffs to sit*, not escaping them. **ε-bound = non-negotiable; everything else is a dial.**

**Ordered leaf-level scan — who has it:**
| Index | Data all at leaf level? | Scan = walk one level? | Quality |
|---|---|---|---|
| B+-tree | yes | yes (sibling-linked) | great |
| **PGM** | yes (sorted array) | yes (walk the array) | **best** — contiguous, pointer-free |
| ALEX | yes (data nodes) | yes (sibling pointers) | good, gap tax → loses past ~1000-key scans |
| LIPP | **no** (scattered, all levels) | **no** (tree-walk) | **poor** — the unified-node price |

*array-backed* (PGM: scan = increment a pointer, cache-best, hard to update in place) vs *linked-node* (B+-tree/ALEX: sibling pointers, updates easier, a hop per boundary). **Design note: keep gaps in the SEGMENT index but data dense → PGM-class scans; gapping the data inherits ALEX's tax (measure it).**

---

## 2. What I like (primitives to borrow) + source

Full catalog in `design-primitives-parts-bin.md`.

- **PGM** — hard ε bound · fully-learned recursive hierarchy (no B+-tree) · optimal convex-hull segmentation · **computed contiguous navigation (pointer-free)** · the free lemma (sub-range monotonicity).
- **ALEX** — **local per-node splitting**, esp. **split-down** (add resolution locally, no parent disturbance) · gapped arrays.
- **LIPP** — the *research pattern* "split on an accuracy event → corrective balancer → proven bound" (existence-proof it's publishable). And: eliminating search is the biggest single lookup win (even if LIPP's *way* is too costly).
- **B+-tree** — **incremental, local, worst-case-bounded rebalancing** (split/merge). The *good* model for a corrective op — opposite of LIPP's bulk rebuild.

---

## 3. What I want to AVOID + who taught me

- **ALEX — no error bound** → degrades under drift. *Avoid:* keep a hard bound.
- **ALEX — reactive cost-model + magic constants** (`dₗ=0.6, dᵤ=0.8`, 50% deviation), "works in our experience"; §4.3.6 admits it mis-predicts under shift. *Avoid:* deterministic trigger, not tuned heuristic.
- **ALEX — splits at the geometric midpoint** (a guess). *Avoid:* cut where the data dictates.
- **PGM — the shelf → `log n` read amplification**, whole-run rebuild for one key. *Avoid:* in-place local edits.
- **LIPP — bulk adjustment → tail spikes + concurrency-hostile** (locks a whole subtree O(n log n)). *Avoid:* local/incremental corrective op.
- **LIPP — global `G` shape assumption** → static, drift-fragile, reintroduces tuning. *Avoid:* no global functional bet.
- **LIPP — space-hungry + data scattered → poor scans.** *Avoid:* watch space (equal-memory) and scan locality.
- **General:** amortized-but-spiky writes; cost-model heuristics standing in for guarantees; hardcoded distribution assumptions. **Recurring lesson: structural guarantee > hope-the-tuning-is-right.**

---

## 4. The gap I'm circling (held loosely)

Nobody has all of these **at once**: *in-place updates + hard worst-case bound + pointer-free navigation + predictable bounded writes.* PGM has bound + pointer-free but pays read amplification. ALEX is in-place but unbounded/heuristic. LIPP removes search but is space-hungry, spiky, concurrency-hostile. **The white space is the intersection** — that's what I'm aiming at, not any one mechanism.

---

## 5. THE CORE CANDIDATE (front-runner — not yet committed)

> Keep ALEX's local in-place split, but make every split **preserve an ε-bound** (trigger = "a key would violate ε"; cut = where segmentation forces it), over a **pointer-free gapped *segment* array**, with a **local B+-tree-style merge** (not LIPP's bulk rebuild) as the balancer. = *ALEX's locality + PGM's guarantee + B+-tree's predictable rebalancing, distribution-agnostic.*

**Why feasible — the free lemma.** Sub-range monotonicity (PGM §2.1): any sub-range of an ε-valid segment is ε-valid → splitting at *any* interior point keeps both halves ε-valid **for free.** "Does splitting preserve the bound?" is already *yes*. (Each half needs a cheap local re-fit, not a run rebuild.)

**The two hard parts:**
- **Hard Part 1 — Layout.** Pointer-free routing needs contiguous segments, but inserting a split segment shifts the array → **gapped segment array + bitmap** (ALEX's trick, one level up). Cost: gap space (quantify).
- **Hard Part 2 — THE THEOREM (the actual contribution).** Local splits erode minimality (drift from optimal `m`). **Prove segment count stays within `c·m` under arbitrary updates, with local merge-on-underflow as the balancer.** The learned-index analog of "B-trees stay balanced." Main scientific risk; even a negative characterization is publishable. *Nobody did this — ALEX never implemented merge; LIPP merged only in bulk.*

**The honest counter I keep answering:** ε bounds *positions*, not *latency* (cache misses aren't in ε). *Resolution:* ε = the **invariant I prove**; latency = the **evidence I measure**.

**Why the bound is the thesis (not the split/layout):** the split is *free* (lemma), the layout is *engineering* (known parts) — the **`c·m` bound under local split+merge is the only piece that's both open AND hard AND uniquely mine.** That's the frontier.

**Kept open:** could instead bound LIPP-style (precise positions + ε fallback), or go full-PGM pointer-free everywhere. Holding until I've read more.

---

## 6. IDEAS LEDGER

*Every idea I've had, tagged. Scan the table; details below.*

| Idea | Status | One-line verdict |
|---|---|---|
| **Proactive ε-proximity splitting** | ⭐ promising | split *before* violating ε (when error nears the bound) → predictable writes + early drift signal, guarantee intact. **Core-adjacent.** |
| Defer *optimization*, never the *guarantee* | ⭐ promising | background-merge to smooth writes, but always split eagerly to keep ε hard. (A4-compatible.) |
| Per-segment adaptive transform `G` | 🔧 optional | space-only, lookup-neutral, complicates merge. Safe (PGM floor). Future-work garnish. |
| Connected spline + per-piece `G` (greedy) | 🔧 optional | smaller (shared knots) but weakens free-split (pinned halves). Documented alternative to disjoint; pick by measurement. |
| Relax to 2ε + dirty-flag + balancer | 🗄️ set aside | collapses into ALEX (soft bound + reactive repair); surrenders the *provable* bound. |
| Shelf-of-RadixSplines | 🗄️ set aside | doesn't remove read amplification (the *shelf* is the problem, not the pile index). |

### ⭐ Promising

**Proactive ε-proximity splitting** *(core-adjacent — a scheduling policy for the core split)*
- **What:** split *early* — while keys are still within ε — when the segment's error is *approaching* the bound (e.g., max or p90 prediction-error > 0.8·ε), instead of waiting for a violation.
- **Why strong:** ✅ never violates ε (guarantee fully intact — splits *before*) · ✅ theorem stays clean (ε-valid → ε-valid) · ✅ does NOT collapse to ALEX (triggers on **ε-proximity = my actual constraint**, not on *cost*) · ✅ serves **predictable writes (A4)** by spreading split-work into calm periods instead of clustering at violations · ✅ it's **proactive** — the reactive→proactive frontier the whole field misses · ✅ ε-crowding is an **early local-drift signal** (keys pressing the bound = the line's slope no longer matches incoming data → split adapts *before* it hurts).
- **🔑 Critical distinction — measure ε-proximity, NOT density.** "Split at 80% of the **ε-budget** (error vs bound)" = the good idea. "Split at 80% **full** (capacity)" = accidentally ALEX's `dᵤ` (wrong currency — my segments are bounded by ε-fit, not slot-count). A segment can be 80% full and ε-fine, or 30% full and ε-saturated.
- **Only my design can do this** — you can only "split before hitting ε" if you *have* an ε to measure against. ALEX doesn't.
- **Costs (real, tunable):** more segments → **larger `c` in the `c·m` bound** (the threshold *tunes* space-vs-predictability — a knob, but a *principled* one in ε-units). Needs a **cheap per-segment error stat** (one running number: max error). **Open Q:** *derive* the threshold (e.g. from insert rate, so expected-inserts-before-violation > split cost) rather than hand-tune it.

**Defer the *optimization*, never the *guarantee*** *(the salvaged kernel of the 2ε idea)*
- **What:** to smooth tail latency, run the expensive *merge/rebalance* (the `c·m`-optimality maintenance) lazily in the **background** — but always do the *split* eagerly (it's cheap — free lemma — and it's what holds ε).
- **Why it works:** ε stays *hard at all times* (splits are synchronous); only *space-optimality* is eventually-consistent. Gives the predictable-writes win **without** making the guarantee contingent on a background process. Clean responsibility split: **ε-violation → split now (holds the bound); drift-from-optimal → merge later (holds the count).**

### 🔧 Optional (future-work garnishes — subordinate to the core)

**Per-segment adaptive transform `G`** — fit `pos = A·G(k)+b` with a *fixed cheap monotone* `G` chosen *per segment* from a menu, to straighten curved regions → fewer segments → smaller index.
- **Preserves the machinery (the non-obvious result, worth keeping):** ε-provable by convex hull run in `(G(k), p)`-space (`G` warps the **x-axis**; ε lives on the **y-axis** — untouched; constraint still linear in `(A,b)`); free split survives (sub-range monotonicity in `G`-space); **greedy stays optimal for count** (union of prefix-closed predicates is prefix-closed). Build = O(m·n) single-pass + a 2-bit tag/segment.
- **⭐ Safety net — worst case = PGM.** `G=x` is always in the menu → a fancy `G` is chosen only if it reaches ≥ as far as linear → **segment count ≤ plain-PGM, always** (pure Pareto improvement; floor = what I'd ship anyway).
- **Menu — cheap monotone only (hot loop!):** ✅ `x², x³`, low-degree poly, bit-shifts. ✅ **cheap log via bit tricks** — `log₂(x) ≈ highest-set-bit` (`clz`/`BSR`, ~1–3 cyc) or reinterpret-float-bits (≈ linear in `log₂`, the fast-inverse-sqrt family). *Works because `G` only needs to be log-shaped + monotone, NOT accurate* — approximation just shifts the fitted line, ε (position axis) unaffected → **heavy-tailed data reachable cheaply.** 🟡 `√x` (borderline). ❌ exact `ln`/`exp`/trig (~20–50 cyc) — but the bit-trick removes the need.
- **Cost-benefit:** ✅ **index size down (the only real win — the ALEX-M-scrutinized axis)** · 🟡 lookup ~neutral (search still `log ε`; `G`-eval + branch *adds* hot-loop cost; **don't sell as a speed win**) · ❌ build ~m× · ❌ **merge complexity up (taxes Hard Part 2)** · ❌ count≠latency proxy breaks if transforms differ in cost · ↔️ guarantee same.
- **Verdict:** minor space-reclamation, lookup-neutral, safe. **Not thesis material** (fit is near-solved; taxes my hardest part). Reserved optional mode, **activate only if space is a *measured* problem.** Keepers: the transform-agnostic ε+greedy proof, the bit-trick-log, and the discipline of scoping it *out*.

**Connected spline + per-piece `G` (greedy, not optimal)** — shared knots → ~1 number/piece (smaller); drop optimality, go greedy.
- **Greedy dissolves the build conflict:** optimal-spline-`G` falls into DP (O(n²·m), not incremental — fatal); greedy = single-pass O(m·n), incremental. ✓
- **Residual tension (precise):** continuity *pins* each split-half at its shared knot → one fewer degree of freedom → re-fitting (old ∪ new ∪ pinned-knot) within ε is **strictly harder, sometimes infeasible** where disjoint succeeds (→ more pieces, or touch a neighbor = non-local). Disjoint's free lemma guarantees local feasibility; spline weakens it to "usually local." *(Structural to continuity — greedy-vs-optimal doesn't change it.)*
- **Tradeoff, not conflict:** disjoint (full line/piece, **guaranteed** local split) vs spline (shared knots = **smaller**, **usually**-local split). Both single-pass.
- **Verdict:** **default = disjoint** (the free-split lemma is the bedrock of the core). Spline+`G`+greedy = legitimate alternative for a *static*/space-critical variant, and possibly fine dynamically **if pinned-half infeasibility is rare on real data — an empirical Q to measure.** Keep both; pick by measurement.

### 🗄️ Set aside (considered, with why — so I don't revisit blindly)

**Relax to 2ε + dirty-flag + background balancer** — insert past ε, mark the segment "needs cleanup," fix later.
- **Why set aside:** this *is* ALEX — soft bound + reactive repair (dirty-flag = cost model, balancer = ALEX's splits). **Surrenders the *provable* worst-case query bound that is my entire differentiator.** Makes the guarantee *contingent on the balancer keeping pace* — a burst/adversary outruns it, pushing keys to 2ε, 3ε, unbounded. Reintroduces the exact fragility I criticize in ALEX/LIPP. Also makes the theorem contingent (balancer-throughput-dependent) instead of structural.
- **Salvaged kernel** → see "defer the optimization, never the guarantee" (⭐). Defer the *merge*, never the *bound*.

**Shelf-of-RadixSplines** — make each PGM-shelf pile a RadixSpline instead of static PGM.
- **Why set aside:** the per-pile index was never the problem — **the *shelf* (many piles → search all → `log n` read amplification) is.** Swapping the pile index doesn't touch the structural `log n`. Collapses to "RadixSpline as an LSM per-file index" (already exists; wrong side of the in-place-vs-rebuild fork). Buys none of my desiderata.
- **Lesson:** *a component swap can't fix a problem that lives in the structure.* My thesis attacks the structure (no shelf), which is why it's the right target.

---

## 7. DESIGN RAZORS (principles I keep rediscovering)

Filters for evaluating new ideas — guides, not vetoes.

1. **The hard ε-bound is non-negotiable.** It's the one thing nobody else has. The moment a key is allowed past ε "temporarily," I'm ALEX with extra steps. **Defer the *optimization*, never the *guarantee*.** Anything that makes "is every key within ε?" depend on "has the balancer caught up?" trades the crown jewel for a systems parameter.
2. **Does this couple the pieces?** My update mechanism lives on pieces being *decoupled* enough to edit one locally. Continuity, shared knots, cross-piece optimization all *couple* → fight locality. **Disjoint segments are the default for a reason.** (Per-segment `G` passed — independent; splines couple.)
3. **Measure ε-proximity, not density.** My segments are bounded by *ε-fit*, not slot-count. Triggers should be in *units of my guarantee* (fractions of ε), not capacity %. (Density % = accidentally importing ALEX's currency.)
4. **Don't over-invest in the fit.** Segmentation is solved (PGM optimal) / near-solved (RadixSpline cheap) / provably-hard-to-improve (free-knot). The novelty is the **update mechanism**, not the fit. Treat fit choices as settled components; aim originality at split/merge/structure.
5. **Be space-honest.** Compare at *equal memory* (the ALEX-M lesson). Don't let gap-space inflate a fake win. Space is the axis the field scrutinizes hardest.
6. **Proactive > reactive, where compatible.** Anticipating (split before violation) beats repairing-after (ALEX cost model, LIPP bulk adjustment) — *if* it preserves the guarantee. The useful "intelligence" is predicting where to split, not a bigger model.
7. **Structural guarantee > hope-the-tuning-is-right.** The thread through every anti-pattern. A bound that holds by construction beats a heuristic that holds in practice.

---

## 8. Still need to read before I commit

Any could change or kill the front-runner:
- **SIGMOD'26 "Updatable Learned Index for Time-Space Tradeoff"** — ⚠️ **SCOOP CHECK, highest priority.** Closest published cousin; did they already prove my theorem?
- **The two robustness papers, in full** (Are-Ready, Robustness Issues) — they *define the bar* + the equal-memory (ALEX-M) methodology. Mandatory.
- **B-tree split/merge amortized analysis** — the *proof template* for the `c·m` bound (treat as a math read; arguably more important than any learned-index paper now).
- **RadixSpline** — *(done — static/LSM, donates the radix-routing primitive; on the wrong side of my fork, doesn't threaten the theorem.)*
- *Then* decide — and only then promote front-runner → thesis.

---

## One-line stance
> *A learned index that is in-place, worst-case-bounded, pointer-free, and predictable — PGM's guarantee + ALEX's locality + B+-tree's rebalancing, avoiding ALEX's heuristics, PGM's read amplification, and LIPP's bulk-rebuild spikes. Core = ε-preserving local split + local merge, feasible by sub-range monotonicity; the contribution is the `c·m` bound. Not committing until the literature's done.*