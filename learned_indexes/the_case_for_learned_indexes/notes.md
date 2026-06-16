# The Case for Learned Index Structures — Notes

*Kraska, Beutel, Chi, Dean, Polyzotis (SIGMOD 2018)*

---

## The One Big Idea

**An index is just a model that predicts where data lives.** Replace the hand-built structure with a learned model that captures the data distribution, and add a small auxiliary structure to restore any hard guarantees the model can't give on its own.

Everything in the paper is this idea applied three ways:

| Index type | Traditional structure | What the model learns |
|---|---|---|
| **Range** | B-Tree | CDF → predicts position in sorted array |
| **Point** | Hash-map | CDF → spreads keys to avoid conflicts |
| **Existence** | Bloom filter | Classifier → separates keys from non-keys |

---

## 1. Introduction

- ML opens the opportunity to **learn a model that reflects the patterns in the data**, enabling automatic synthesis of specialized index structures ("learned indexes") with low engineering cost.
- No current indexes take advantage of the data distribution — they're general-purpose and assume nothing about the data.
- Knowing the exact distribution lets you highly optimize almost *any* index structure.
- Indexes are *already* largely learned models, so swapping in other ML models is surprisingly natural:
  - A **B-Tree** takes a key and predicts the position of a record in a sorted set.
  - A **Bloom filter** is a binary classifier predicting whether a key exists.
- **Hardware tailwind:** every CPU has SIMD, and GPUs/TPUs are becoming ubiquitous → the cost of executing a model may become negligible. *(Paper is from 2018; in 2026 this prediction landed — NPUs ship in essentially every flagship phone and "AI" laptop chip. The one catch the paper flagged, invocation latency, is still the live concern.)*
- **Main contribution:** outline + evaluate a novel approach that *complements* existing work. Key observation: **many data structures decompose into a learned model + an auxiliary structure** that provides the same semantic guarantees.
- The power comes from using **continuous functions describing the data distribution** to build more efficient structures/algorithms.
- Open challenges remain — notably **write-heavy workloads**.

---

## 2. Range Index

- A B-Tree over a sorted primary key maps a lookup key → position, guaranteeing the record at that position has the **first key ≥ the lookup key**.
- The B-Tree is effectively a **regression tree**: maps a key to a position with min-error 0 and max-error = page size, guaranteeing the key is findable in that region if it exists.
- We can swap in any ML model (incl. neural nets) **as long as it provides similar min/max error guarantees**.
- The guarantee only holds over **stored** keys, not all possible keys. New data → B-Trees rebalance; models **retrain**.
- **Crucially, the strong error bounds aren't even needed.** Data is sorted anyway, so any error is corrected by a **local search** around the prediction (e.g. exponential search) — which even allows **non-monotonic models**.
- Simplifying assumptions for the paper: in-memory, dense, sorted array. Disk-resident / fine-grained paging / insert-heavy → more research needed.

> **Aside — what about skip lists?** Skip lists are a natural fit to ask about, since they're another ordered structure that "jumps roughly to the right region." There *is* work here (e.g. **FineStore-SL**, 2026) that swaps a skip list in as the backbone, with a learned model on top. The appeal: skip lists are **efficient for concurrent updates**, which is exactly the weakness of array-based learned indexes. So the model does the cheap "jump to the region" work while the skip list handles mutation/concurrency. It's a niche branch though — the mainstream successors (ALEX, PGM) went the updatable array/tree route instead.

### 2.1 What Model Complexity Can We Afford?

- A B-Tree with page size 100 over 100M records has a **precision gain of 1/100 per node**, traversing log₁₀₀(N) nodes (100M → 1M → 10k → …).
- Traversing one B-Tree page via binary search ≈ **50 cycles**, and is **hard to parallelize**.
- A modern CPU does **8–16 SIMD ops/cycle** → a model wins as long as its precision gain beats 1/100 per **~400 arithmetic ops** (50 × 8).
- That calc assumes pages are cached; a single **cache miss = 50–100 extra cycles**, allowing even more complex models.
- **ML accelerators change the game** — more complex models in the same time, offloading the CPU.
  - *(Paper's example)* NVIDIA Tesla V100: 120 TFLOPs (~60k ops/cycle). If the index fits in GPU memory, ~1M neural-net ops in just 30 cycles.

### 2.2 Range Index Models *are* CDF Models

- **Key observation:** a model predicting position in a sorted array is approximating the **cumulative distribution function (CDF)**:

  **p = F(Key) × N** — where `p` = position estimate, `F(Key)` = estimated CDF = P(X ≤ Key), and `N` = total keys.

- Implications:
  1. Indexing literally **requires learning a data distribution** (a B-Tree "learns" it as a regression tree; linear regression learns it by minimizing squared error).
  2. Distribution estimation is well-studied → learned indexes inherit decades of research.
  3. The CDF idea also optimizes **other** index types and algorithms (foreshadowing).
  4. Long history on how closely theoretical CDFs approximate empirical ones → a foothold for theory.

### 2.3 A First, Naïve Learned Index

- 200M web-server log records; secondary index over timestamps; 2-layer FC net, 32 neurons/layer, ReLU, in TensorFlow.
- Result: ~1250 predictions/sec ≈ **80,000 ns** per model execution.
  - vs **B-Tree traversal ≈ 300 ns**, **binary search ≈ 900 ns**. (Naïve approach loses badly.)
- **Why it's slow:**
  1. **TensorFlow overhead** — built for large models, big invocation cost (worse with Python front-end).
  2. **B-Trees overfit cheaply** via simple if-statements. Models capture the general CDF shape efficiently but struggle with the **"last mile"** — narrowing from thousands → hundreds costs a single net a lot of space/CPU.
  3. **B-Trees are cache/op-efficient** (top nodes stay cached); standard nets need *all* weights for every prediction → many multiplications.

---

## 3. The RM-Index

Three pieces: the **Learning Index Framework (LIF)**, **Recursive Model Indexes (RMI)**, and **standard-error-based search**. Focus is simple FC neural nets for their simplicity/flexibility.

### 3.1 The Learning Index Framework (LIF)

- An **index synthesis system**: given a spec, it generates configs, optimizes, and tests them automatically.
- **Two-phase trick:** trains with TensorFlow when needed, but **never uses TensorFlow at inference**. It extracts the trained weights and **code-generates lean, dependency-free C++** for the actual lookups.
- Result: **80,000 ns → ~30 ns.** Stripping the general-purpose framework is what makes learned indexes fast enough to beat a B-Tree.
- *Caveat:* LIF is a **research prototype**, not production-grade — extra counters, virtual calls, no hand-tuned SIMD. Fair for their comparisons (same tax on everyone) but real numbers could be even better.

### 3.2 The Recursive Model Index (RMI)

- **Core problem:** one model alone can't get accurate enough. Going 100M → 100 in one shot is hard; 100M → 10k is easy.
- **Solution:** a **hierarchy of small "expert" models**. Each stage takes the key, makes a prediction, and that prediction **directly picks the next model** — until the final stage predicts the position.
  - The handoff math: `⌊ M · f(x) / N ⌋` scales a model's output to an index into the next stage's models.
- **Not a tree:** different parents can point to the same child, and models don't cover equal-sized chunks.
- **Benefits:** (1) decouples model size from execution cost; (2) the overall CDF shape is easy to learn; (3) divide-and-conquer makes last-mile accuracy cheap; (4) **no search between stages** → can be expressed as one big matrix multiply (GPU/TPU-friendly).

> Turns the impossibly-hard "find the exact position in 100M records" into a chain of easy narrow-it-down steps, with no searching in between — accurate *and* fast.

### 3.3 Hybrid Indexes

- **You don't have to use the same model everywhere — mix and match per layer.**
  - **Top layer:** small ReLU neural net (captures the complex overall shape).
  - **Bottom layers:** thousands of cheap **linear regressions** (tiny, fast — all the easy last-mile needs).
  - **Too-weird-to-learn chunks:** fall back to a plain **B-Tree**.
- Training cascades the data downward: each stage trains, then routes its records into the right next-stage bucket.
- At the bottom, any model whose **max error > threshold** is **automatically replaced with a B-Tree**.
- **Worst-case guarantee:** if the data is pathological, *every* model becomes a B-Tree → it degrades gracefully to "no worse than a B-Tree."
- Min/max error is stored **per bottom-stage model**, so each gets its own tailored search window.

### 3.4 Search Strategies & Monotonicity

- The model gets you **close**; you still need a **last-mile search** to find the exact key.
- Range indexes offer `upper_bound(key)` / `lower_bound(key)`.
- Setup: normally plain **binary search wins** — fancy search rarely beats it (less overhead).
- **Learned edge:** a B-Tree only narrows you to a *page*; a model predicts the *actual position*. Two strategies exploit this:
  - **Model Biased Search:** binary search, but the first midpoint = the model's prediction.
  - **Biased Quaternary Search:** probes **three** points at once (`pos − σ`, `pos`, `pos + σ`). Hardware **pre-fetches all three in parallel**, hiding memory latency when data isn't cached. Bets the answer is near `pos`.
- Both use the prediction to **start in the right place**.
- Search window = the stored **min/max error** per bottom model (run every key, record worst over/under-shoot).
- **Monotonicity matters for *missing* keys.** A monotonic model guarantees "bigger key → bigger-or-equal position," so the search window around a missing key is trustworthy. Non-monotonic models can return the wrong neighbor. Fixes: **force monotonicity**, **widen the window at its edges**, or **exponential search** (no stored errors needed).

### 3.5 Indexing Strings

- Everything so far assumed numeric keys — but a model can't do math on `"apple"`.
- **Step 1 — Tokenization:** each character → its ASCII value → a *vector* of numbers.
- **Step 2 — Fixed length:** pick max length `N`; truncate longer strings, zero-pad shorter ones.
- **Step 3 — Same architecture, bigger input:** the same RMI net hierarchy, but input is a vector not a scalar.
- **The catch:** work now scales with `N` (linear ≈ N mults; net ≈ h × N) → string indexing is **costlier**, wins over B-Trees **smaller**.
- **Step 4 — Future work / scratching the surface:**
  - Smarter tokenization (e.g. **wordpieces** from NLP — meaningful chunks instead of raw chars).
  - **Suffix-trees + learned models.**
  - Sequence-native architectures (**RNNs, CNNs**).

### 3.6 Training

- Building a learned index is **fast**: cheap linear models train in a **single pass via a closed-form formula**; small nets converge in **a few passes**.
- A **200M-record index trains in seconds** (more with auto-tuning); bigger nets take minutes per model.
- The top model can **stop early** — it only needs rough accuracy.
- *Note:* this is all **static, read-only build time**. Says nothing about **updating** the index on new data — the hard part.

### 3.7 Results

- **Datasets:** Weblogs (messy human timestamps — near worst case), Maps (smooth longitude — easy), Lognormal (synthetic, non-linear).
- **Integers:** learned index beats even a **tuned production B-Tree** — **up to 3× faster, ~100× smaller** — and beats the specialized baselines (Lookup-Table, FAST, fixed B-Tree) on the combined size/speed front.
  - FAST is competitive on speed but huge (1024 MB) due to power-of-2 alignment.
  - Quantization (4/8-bit params) could shrink learned indexes even further.
- **Strings:** wins shrink — model execution + string comparisons are both pricier. This is exactly where **hybrid models** and **smarter search** (quaternary 658 ns vs binary 1102 ns for the same net) earn their keep.

> **Pattern:** learned indexes win biggest when the data is **learnable** and comparisons are **cheap**; the gap narrows when either gets harder. A consistent, sensible trend — not a magic bullet.

---

## 4. Point Index (Hash-Model Index)

- Switching from ranges to **point lookups**: B-Trees find ranges; a hash-map answers *"is key X here, and where?"*
- **Core problem: conflicts (collisions).** Even a *perfectly random* hash with #slots = #records gives **~33% conflicts** (~33M of 100M) — the **birthday paradox**. Every conflict = extra work (e.g. linked-list chaining → cache misses).
- **Insight:** conflicts come from **uneven spreading**. A model that knows the distribution spreads keys *more evenly* than randomness:

  **h(K) = F(K) × M** — the **same learned CDF**, scaled to table size `M`. Perfect CDF → zero conflicts.

- **Caveats:**
  - Only helps if data is **non-uniform** (uniform data → a random hash is already as good).
  - Payoff is **architecture-dependent** — learned hashing is orthogonal, plugs into any hash-map type; worth it only if conflicts are expensive in your setup.

### 4.2 Results

| Dataset | Random hash | Learned | Reduction |
|---|---|---|---|
| Map | 35.3% | 7.9% | **77.5%** |
| Web | 35.3% | 24.7% | 30.0% |
| Lognormal | 35.4% | 25.9% | 26.7% |

- **Map wins biggest** (model learns its smooth CDF accurately); messier data → smaller wins. Same "learnable = better" theme.
- **Large payloads / separate chaining:** big win (up to **80% less wasted storage**, +13 ns — fewer conflicts also means fewer cache misses).
- **Tiny payloads:** not worth it — Cuckoo hashing still wins.
- **Distributed settings:** where it shines. In NAM-DB, a conflict = an extra **RDMA round-trip (microseconds)**, so model cost (nanoseconds) is negligible and any conflict reduction is a big win.

> The same CDF model, repurposed: range = predict position, point = spread keys to avoid conflicts.

---

## 5. Existence Index (Bloom Filters)

- Bloom filters are space-efficient but **memory-hungry at scale**: 1B records ≈ **1.76 GB**; at 0.01% FPR ≈ **2.23 GB**.
- **Bloom filter recap:** bit array + k hash functions. Insert = set k bits to 1. Check = if any of the k bits is 0 → **definitely not present**; all 1 → **probably present**.
  - **No false negatives, possible false positives.** "No" is certain, "yes" is a maybe.
- **Insight:** a normal filter treats keys as random — learns nothing about *what makes a key a key*. But if structure separates keys from non-keys, a model can **learn the boundary** and shrink the filter. (Extreme case: keys = integers 0..n → just check `0 ≤ x < n`.)
- **What "good" means here flips vs. point indexes:** a point hash wants **few collisions among keys**; a Bloom function wants **many collisions among keys, many among non-keys, but few *between* the two** (a separation problem).

### 5.1.1 Approach 1 — Bloom Filter as Classification

- Train a model `f(x)` outputting **P(x is a key)**, on **both keys and known non-keys** (hence you need a non-key set). For strings → RNN/CNN, sigmoid output, log loss.
- Pick a threshold τ: `f(x) ≥ τ` → "key."
- **The danger:** the model has false positives (fine) *and* **false negatives (forbidden)**.
- **The fix — overflow Bloom filter:** collect all real keys the model wrongly rejects (`f(x) < τ`) into a small traditional Bloom filter. Lookup:
  1. `f(x) ≥ τ` → key exists.
  2. Else → **don't trust the model**, check the overflow filter.
- This is the **model + auxiliary structure** decomposition again. The overflow filter only holds the model's *misses*, so it's small (its size scales with the FNR).

### 5.1.2 Approach 2 — Model as a Hash Function

- Use the model's output as a Bloom hash: **d = ⌊f(x) × m⌋**. Since it's trained to separate keys from non-keys, it maps most keys to the high end of the bit array, non-keys to the low end.

### 5.2 Results — Phishing URLs

- 1.7M blacklisted URLs; non-keys = random + whitelisted look-alikes; small char-level **GRU**.

| Target FPR | Normal filter | Learned (model + overflow) | Saving |
|---|---|---|---|
| 1.0% | 2.04 MB | — (model itself only 0.0259 MB) | — |
| 0.5% | — | 1.31 MB (FNR 55%) | **36%** |
| 0.1% | 3.06 MB | 2.59 MB (FNR 76%) | 15% |

- **Trade-off:** lower target FPR → higher FNR → bigger overflow filter → smaller savings.
- **Covariate shift:** random-only non-keys → 60% saving; whitelisted-only (confusable) → 21%. The more confusable the non-keys, the harder the job.
- **Big perk:** the model can use **richer features than the filter** (WHOIS, IP, page signals) → better accuracy, smaller index, *still* no false negatives.

> Trains a model to predict "key vs non-key," then pairs it with a small overflow filter holding only its false negatives — restoring the sacred no-false-negatives guarantee while shrinking memory.

---

## 6. Related Work

Recurring refrain across every category: *prior work optimizes structure or encoding, but none of it learns the data distribution.*

- **B-Trees & variants** (B+-trees, T-trees, red-black, cache-conscious CSB+-tree, SIMD/GPU FAST, tries/radix-trees): all **orthogonal** — they tune the *structure*, not the *distribution*. Could potentially *combine* with learned models (cf. hybrid indexes).
- **Index compression** (prefix/suffix truncation, dictionary compression, hot/cold): learned indexes compress via a *radically different route* — the data distribution — possibly changing the complexity class (O(n) → O(1)). Complementary: dictionary compression is essentially **embedding** (string → unique int).
- **Closest prior work — A-Trees, BF-Trees, B-Tree interpolation search:**
  - *BF-Trees* — B+-tree with Bloom-filter leaves; don't approximate the CDF.
  - *A-Trees* — piecewise-linear functions to cut leaf count (closest in spirit).
  - *Interpolation search* — guesses position within a page.
  - Learned indexes go further: **replace the entire structure**.
- **Sparse indexes** (Hippo, Block Range Indexes, SMAs): store value-range info but ignore the distribution.
- **Learning hash functions for ANN / LSH:** opposite goal — LSH wants to *group similar items together*; a point hash wants to *spread keys apart*. Doesn't transfer.
- **Perfect hashing:** closest relative for hash-maps (also avoids conflicts), but uses no learning and its **size grows with the data** — a learned hash can be size-*independent*. Also doesn't help B-Trees or Bloom filters.
- **Bloom filters:** builds on classic work but with a different optimization angle (enhanced classifier or model-as-hash).
- **Succinct data structures** (wavelet trees, rank-select): optimize for **H₀ entropy** (bits to *encode*); learned indexes learn the *distribution* to predict positions → could compress **below H₀**, possibly at the cost of slower ops. Also: succinct structures are hand-crafted per use case; learned indexes **automate** via ML. May offer a theoretical framework for studying learned indexes.
- **Modeling CDFs:** the foundation everything rests on; how best to model a CDF is **still open**.
- **Mixture of Experts:** the ancestry of RMI. Key property — **decouples model size from computation cost** (huge ensemble, but only a few experts run per lookup).

> Structurally, this section re-proves the thesis by elimination: since nothing existing uses the data distribution, "indexes as learned CDF models" really is the new idea.

---

## 7. Conclusion & Future Work

- **Thesis restated:** learned indexes help by **exploiting the data distribution**.
- **Other ML models:** they only tested linear models + small nets — lots of other model types and combinations to explore.
- **Multi-dimensional indexes** *(called the most exciting direction)*: real queries filter on many attributes at once, which is genuinely hard traditionally. NNs excel at high-dimensional relationships → a model predicting position from *any combination of attributes* could be huge. *(This became real — Flood, Tsunami.)*
- **Beyond indexing — learned algorithms:** the CDF trick generalizes to **sorting and joins**. Sorting idea: run records through the CDF to drop each near its final spot → "nearly sorted" → cheap cleanup (insertion sort). *(This became "Learned Sort.")*
- **GPU/TPUs:** make learned indexes cheaper, and they'll *fit* (great compression). The honest catch: **invocation latency (2–3 µs)** dwarfs a nanosecond lookup. Rebuttal: improving CPU–accelerator integration + **batching** amortizes it away.
- **Summary:** learned models can beat state-of-the-art indexes — a fruitful research direction.

> The lasting contribution isn't a benchmark number — it's the **reframing**: "an index is a model that predicts a position." That single idea spawned years of follow-up work (multi-dimensional indexes, learned sort, updatable variants).

---

## 8. Research Angles & Opportunities

*Combining the paper's stated future work with the threads I found interesting while reading.*

### Stated by the paper
- **Multi-dimensional learned indexes** — predict position filtered by *any* combination of attributes. The flagged "most exciting" direction; plays directly to NN strengths. *(Partly realized: Flood, Tsunami.)*
- **Learned algorithms beyond indexing** — CDF-accelerated **sorting** and **joins**. *(Realized: Learned Sort.)*
- **Other ML model types** — beyond linear + small nets; new architectures and hybrid combinations with classical structures.
- **Write-heavy / update handling** — the biggest open gap. The whole paper is static & read-only; nothing addresses inserts/updates. *(This is the gap that ALEX and the PGM-index were built to fill — worth reading next.)*
- **Disk-resident & fine-grained paging** — their techniques translate to large-block disk data, but fine-grained paging / insert-heavy needs more work.
- **GPU/TPU deployment** — how to actually run learned indexes on accelerators given invocation latency; batching strategies.

### My own threads from the notes
- **The "last-mile" problem** — the recurring weak spot: models nail the general CDF shape but struggle to pin down individual positions (thousands → hundreds). Is there a better model class / architecture purpose-built for the last mile? (RMI's bottom-stage linear models are the current answer; could be improved.)
- **Better string tokenization** — replace one-number-per-character with NLP-style **wordpieces** (meaningful chunks). Could shrink models and cut the per-string compute cost that currently makes string indexing weak.
- **Suffix-trees + learned models** — an unexplored combination for string keys.
- **Sequence-native architectures for strings** — RNNs/CNNs built for sequences may model string keys far better than feed-forward nets.
- **Production-grade LIF** — the framework is a research prototype (extra counters, virtual calls, no hand-tuned SIMD). A clean, SIMD-optimized implementation could push the reported numbers further.
- **Skip-list-backed learned indexes** — pairing a learned model with a skip list to get concurrency/update support that array-based learned indexes lack. *(Exists but niche — FineStore-SL; could be pushed further vs. the mainstream ALEX/PGM line.)*
- **Beating H₀ entropy** — the succinct-data-structures connection hints learned indexes might compress *below* the entropy bound by exploiting distributional structure. Theoretically interesting and largely open.
- **Richer features for learned Bloom filters** — since the model needn't share features with the filter, what's the ceiling on filter shrinkage if you feed in auxiliary signals (WHOIS, IP, etc.)? And how to handle **covariate shift** in query distributions principledly (importance weighting, adversarial training).