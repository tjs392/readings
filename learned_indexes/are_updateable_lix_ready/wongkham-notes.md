# Wongkham notes — *Are Updatable Learned Indexes Ready?* (the eval-side adversary)

**Wongkham, Lu, Liu, Zhong, Lo, Wang. PVLDB 15(11):3004–3017, 2022.** (CUHK + Simon Fraser.) Benchmark suite **GRE**, open-source (github.com/gre4index/GRE).

**Where it sits:** core-list paper #5, *read before benchmarking*. Correctly placed. **Yang tells me what the *theorem* must beat; Wongkham tells me what the *benchmark* must beat.** It is simultaneously my eval harness (adopt GRE's grid) and my reviewer's armory (every "learned indexes aren't ready" line a referee will quote comes from here). Read with both hats on.

**One-line relevance:** the paper's *dataset-hardness metric is literally my `m` and my `ε`* — so my `M ≤ 2m−1` is competitive against the exact number on every axis of every heatmap in this paper.

---

## 1. The four things that matter for me (the rest is surface)

**① The hardness metric = my `m(S,ε)`.** Their ε-approximate definition (p.3007, `|F(kᵢ) − rᵢ| ≤ ε`) is my ε-validity; their hardness `H` = segments in the *optimal* PLA = my `m(S,ε)`; both cited to PGM `[17]`, same source as my free lemma. It splits along two granularities I answer separately:
- **local hardness** (ε=32, "challenges model accuracy") → my hard ε-invariant nails this *by construction* (every key within ε, always).
- **global hardness** (ε=4096, "challenges structure + the cost models governing SMOs") → my `c·m` bound + computed routing answer this structurally (no cost model in the maintenance loop).
I have a structural answer to *both* their dimensions, expressed in *their* coordinates. This is the single most useful thing in the paper.

**② ALEX-M = the equal-memory law (the most important page, §5 / Fig 9).** Detune ALEX fill factor 0.7 → 0.2–0.25 to match LIPP's memory, and ALEX-M then *dominates* LIPP on lookups — flipping a published LIPP result. Lesson = my **R6**: compare at equal memory or PGM/ART/ALEX-M eats me. My structure spends memory on the *same* thing ALEX-M does (gaps + slack), so equal-memory isn't just fair, it's the regime I live in. A stock-ALEX comparison is a trap in **both** directions.

**③ The deletion finding is blind to the axis I own (§4.4, Message 8).** "Deletes are lightweight because no model pollution" measures delete *throughput*, not segmentation *health*. None of the five maintains a merge invariant — and they had to *implement deletion themselves* for ALEX+/LIPP+ to benchmark at all. So my silent-reducibility result (deletes push `M` above `2m` with nothing firing) is **invisible to this benchmark** — uncontested, but I must build the metric *and* the delete-side adversary myself (neither exists in the literature).

**④ The landmine: "most real datasets are easy" (Message 2, Lesson 2).** The single most dangerous sentence for me. A referee weaponizes it: *"the field's own benchmark says hard data is rare and semi-artificial (osm = 1-D projection, fb = upsampled) — why optimize a corner that doesn't occur?"* **Rebuttal lives in the same paper → §6.2.** (full defense in §4 below.)

---

## 2. Take / reject ledger

**✅ TAKE**
- **The PLA hardness metric as my eval x-axis.** Report results *against* `H_PLA(ε=32)` and `H_PLA(ε=4096)` — speak the field's language; my bound is stated in it.
- **The GRE harness + dataset battery.** 10 real datasets, the read-only→write-only workload ladder (0/20/50/80/100% write), the heatmap presentation. Reuse, don't reinvent.
- **The equal-memory discipline (ALEX-M).** Always benchmark vs ALEX-M, never stock ALEX. Bake into the eval plan as non-negotiable.
- **Their robustness framing.** Tail latency + distribution-shift as *first-class axes*. These are exactly where the invariant pays.
- **Fig 12 as a ready-made demo.** Their own distribution-shift experiment is my robustness showcase (see §4).

**❌ REJECT / DON'T COMPETE**
- **Average-case throughput on easy data.** Their crown for ALEX+/LIPP. My convex-hull last-mile leaves larger *typical* residuals than least-squares; L1 residual-shaping closes most, not all. Don't headline this.
- **Memory.** Learned ≤3.2× (most-generous framing) and *all* lose to HOT. I spend space on gaps + slack + the ≤2m mapping table. Never sell on space.
- **Raw segment count on a static snapshot.** PGM's from-scratch optimality = exact `m` < my 2m, by construction. (Already in PROJECT-CONTEXT §11; Wongkham reinforces it.)
- **Concurrency as a headline (yet).** See referee-risk §5.

**⚠ NOTE (not take/reject, but carry):**
- Their PGM verdict is a critique of the **shelf**, not of convex-hull segmentation + computed routing — i.e. of the machinery I *discard*. Turn this into a clarification, not a liability (§5).

---

## 3. The equal-memory eval-harness checklist (R6, operationalized)

Before any benchmark number leaves the building:
1. **Equal memory.** Tune fill/density so my index ≈ the baseline's bytes (ALEX-M-style). Report the memory each index actually used alongside throughput.
2. **End-to-end size.** Count the leaf layer (key-position pairs), not just the model hierarchy — Message 9's whole point. My ≤2m mapping table + PMA blocks + gaps all count.
3. **Evaluate where the invariant pays**, not average-case: adversarial inserts (Yang's duplicate pile), tail latency (99.9% + variance, their Fig 10–11 method: sample 1% of ops), distribution shift (their Fig 12 protocol), worst-case after churn.
4. **One dynamic structure**, no rebuild snapshot. The axes where not-rebuilding pays: read-amplification, tail, write-predictability.
5. **Hardness-stratified.** Plot against `H_PLA`; show the *hard-data ≥50%-write* quadrant explicitly (Message 3's red corner — the one I claim).
6. **Report the premium honestly.** Frame = "small bounded average-case premium for a worst-case guarantee they provably can't offer," never "matches ALEX average-case."

---

## 4. ⭐ The Figure-12 rebuttal (defusing "most real datasets are easy")

**The attack (Message 2 / Lesson 2):** real data is easy; hard data is rare and semi-artificial; so a worst-case bound optimizes a corner that doesn't occur.

**The rebuttal, from Wongkham's own §6.2 / Fig 12:** their distribution-shift experiment bulk-loads easy `covid`, then inserts hard `osm` — and ALEX throughput drops **up to 52%** (and PGM/LIPP are sensitive too). This is **easy data *becoming* hard at runtime, non-adversarially.** So "hard data is rare in a static sample" is contradicted by the same paper's *dynamic* result. There is also an asymmetry (hard→easy can *improve* ~15%, Message 11), which only sharpens the point: the cost is paid exactly when the world hardens under you.

**Compose with two existing arguments (PROJECT-CONTEXT §9–10):**
- **kind-not-degree:** a hard invariant covers the tail you *can't* sample — drift (Fig 12) or an adversary (Yang) constructs it; an average-case index has no answer to inputs it didn't see.
- **composability:** a hard-bounded component can anchor a *guaranteed* system; an average-case one cannot. (Why the thesis matters beyond itself.)

**Move:** turn *Wongkham's Fig 12 against Wongkham's Lesson 2*, explicitly, in the writeup. Don't leave Lesson 2 unanswered — a referee will reach for it first.

---

## 5. Referee-risk pre-load

- **🔴 Concurrency (the real exposure).** Lesson 3 = "concurrency + robustness must be first-class, not bolted on." My plan defers concurrency. Defense is real — field precedent (PGM core single-threaded; ALEX+/LIPP+ were *separate* works, done by *these* authors), and ready-*by-construction* (bounded-local writes, the ≤2m mapping table as the CAS publish point — concurrent-design-analysis.md). **Frame as "designed-for, evaluated-later," never "deferred."** This paper makes concurrency a default review axis; keep the paragraph loaded.
- **🟡 "Why not maintain optimal `m`?"** Already answered (boundary-displacement shield, compass §7). Wongkham doesn't add to this but a Wongkham-trained referee asks it.
- **🟢 PGM-base critique = shelf critique (turn into a clarification).** Wongkham dismisses PGM (lookups dominated; wins only pure-insert via LSM buffering; dropped from the main heatmap). But that verdict is on the *run-merge shelf* — exactly what my "no rebuild" identity discards. State plainly: *Wongkham indicts the machinery I don't keep (the shelf → log-n read-amp); I inherit the bound and the computed routing, not the shelf.* A seeming negative for the base becomes a sharpening.

---

## 6. Scorecard — how the thesis stacks up against Wongkham's verdicts

| Axis | Wongkham's verdict | Me |
|---|---|---|
| Read lookups, easy data | ALEX/LIPP win (Msg 1,4) | tie-to-slightly-behind (hull last-mile > least-squares typical residual; L1 closes most) |
| Memory | learned ≤3.2×, all worse than HOT (Msg 9) | **don't compete** — gaps + slack + ≤2m table cost space |
| Writes, easy data | learned competitive | competitive (bounded local split, no cost-model thrash) |
| Writes, **hard data ≥50%** | learned **lose** (Msg 3) | **my quadrant** — `c·m` caps the cascade; no 16MB-bounded amplification; no OOM |
| **Tail latency** | ALEX/LIPP spike on hard (SMOs); ART/HOT/B-tree impeccable (Msg 10) | **my headline** — ε caps the search window by construction → aim at the traditional-index robustness class while staying learned |
| **Distribution shift** | ALEX −52% (§6.2) | should not regress — invariant is distribution-agnostic; Fig 12 = my demo |
| Range scan | learned good; unified-node bad (Msg 12) | PGM-class scans IF data dense at one level (compass §3) — keep gaps in the segment index, data dense |

**The pattern:** I lose-or-tie on every axis Wongkham crowns ALEX+ on (average-case throughput on easy data, memory), and I win on every axis where the paper shows the *whole field* failing (hard-data writes, worst-case tail, shift robustness) — **but with a guarantee none of them carry.** That *is* the thesis. The paper makes my framing mandatory: a small bounded premium for a guarantee ALEX provably can't offer.

---

## 7. Verify / risk

- **⚠ verify — `planet`-style global deflection.** `planet`'s CDF has a sharp kink at ~1M (PGM-balanced over-traverses the sparse side; p.3008). Confirm my *inner routing* blunts this to an arithmetic cost (computed, depth-independent) and doesn't quietly reintroduce PGM's balanced-over-sparse tax before claiming the win.
- **⚠ risk — eval surface size.** Their grid (10 datasets × 5 workloads × concurrency) × my invariant-pays axes is large. Scope hard: adversarial/tail/shift on **one** dynamic structure at equal memory. Resist the easy-corner average case — that's their turf and chasing it costs the guarantee.
- **Open thread:** `wiki` has *duplicate* keys (p.3007) — a free real-world stress for my uniquifier precondition. Worth a targeted run.

---

## 8. Verified citations (checked against the PDF — page = PVLDB page number)

- **Messages 1–12:** 80% single-core win (p.3009); most data easy (3009); learned lose only on hard ≥50% writes (3009); win read-only→read-intensive regardless of hardness (3009); LIPP node-chaining < ALEX key-shifting write-amp (3010); some single-threaded (ALEX) scale post-parallelization, some (LIPP) don't (3010); HT/NUMA affect scalability (3010); deletes lightweight, no model pollution (3011); ≤3.2× space, all worse than HOT (3012); except XIndex low tail latency, ART/HOT/B-tree impeccable (3013); learned sensitive to shift, traditional not (3014); learned good on range, unified-node bad (3014).
- **ALEX-M:** fill factor 0.2–0.25 vs 0.7; dominates LIPP on lookups; LIPP mem 4–5× ALEX (§5 / Fig 9, p.3012).
- **3.2×** = PGM (most-efficient learned) vs ART (least-efficient traditional); all learned > HOT (p.3012).
- **PGM** excluded from heatmap; wins only pure-insert via LSM buffering, "not its core design" (p.3009).
- **Distribution shift:** ALEX −52% (covid→osm), +15% (osm→covid) (§6.2 / Fig 12, p.3014).
- **Hardness:** ε-approximate def + `H` = optimal-PLA segment count, linear-time via PGM `[17]` (§3.2, p.3007); ε=4096 global / ε=32 local, empirically chosen (p.3008).
- **osm** = 1-D projection of spatial; **fb** = upsampled; **wiki** = has duplicate keys (p.3007).
- **Deletion** support: extended ALEX's, implemented LIPP's, for the study (§4.4, p.3010).
- **Verdict:** ALEX+ "almost ready"; LSM-style for write-heavy; hardness-conscious use (§9, p.3015).
- **Hardware:** quad-socket 4× 24-core Xeon Platinum 8268, 96 cores, 768GB (p.3008).

*No claim above is paraphrased from memory — all traced to the uploaded PDF. The take/reject and scorecard rows are my synthesis, not the paper's claims.*
