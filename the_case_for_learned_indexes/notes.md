Learned Indexes

- ML opens up the opportunity to learn a model that reflects the patterns in the data

- No current indexes take advantage of the distribution of the data

- Knowing the exact data distribution enables highly optimizing almost any index structure

- ML opens up the opportunity to learn a model that reflects the patterns in the data and thus to enable the automatic synthesis of specialized index structures, termed learned indexes, with low engineering cost

- Indexes are already to a large extent learned models, making it surprisingly straightforward to replace them with other types of ML models. 

- For example, a B-tree can be considered as a model which takes a key as an input and predicts the position of a data record in a sorted set (the data has to be sorted to enable efficient range requests)

- A bloom filtrer is a binary classifier, which based on a key predicts if a key exists in a set or not.

- Significant benefits especially on the next generation of hardware

- Every CPU already has powerful SIMD capabilities and we speciulate many laptop sand mobile phones will soon have GPU or TPUs (this paper is from 2019, and with the advent of AI this is becoming way more prevalent.)

- As a result, the high cost to execute a neural net or other ML models might actually be negligible in the future. 

- The main contribution of this paper is to outline and evaluate the potential of a novel approach to build indexes, which complements existing work and opens up an entirely new research direction for a decades old field. This is based on the key observatoin that many data structures can be decomposed into a learned model and an auxiliary structure to provide the same semantic guarantees

- The potential power of this approach comes from the fact that continuous functions, describing the data distribution, can be used to build more efficient data structures or algorithms

- Open challenges remain such as how to handle write heavy workloads

Range Index

- Consider a B-Tree index in an anlytics in-memory data (readonly) over the sorted primary key column. In this case, the BTree provides a mapping from a look up key to a position inside the sorted array of records with the guarantee that the key of the record at that position is the first key equal or higher than th elookup key

- My question: what about skip lists? (insert what claude gave me here for my question)

- Here we only assume fixed length records and logical paging over a continuous memory region, ie, single array, not physical pages which are located in different memory regions

- Thus the BTRee is a model, or in ML temrinology, a regression tree: it maps a key to a position with a min and max error (min error 0, max error page size), with a guarantee that the key can be found in that region if it exists. We can replace the index with other types of ML models, including neural nets, as long as they are also able to provide similar storng guarantees about the min and max error

- The BTree only provides the strong min and max error guarantee over the stored keys, not possible keys. For new data, b trees need to be rebalanced or in machine learning terms: retrained, to still be able to provide the same error guarantees.

- Second, and most importantly, the strong error bounds are not even needed. The data has to be sorted anyway to support range requests, so any erorr is easily corrected by a local search around the prediction: eg exponential search. and thus even allows for non monotonic models

- BTrees have a bounded cost for inserts and lookups and are particularly good at taking advantage of the cache

- Also, BTrees can map keys to pages which are not continuously mapped to memory or disk

- For most of the discussion in this paper, we keep the simplified assumptions of this section: we only index an in memory dense array that is sorted by key.

- While some of our techniques translate well to some scenarios (disk resident data with very large blocks, for example), for other scenarios (fine grained paging, insert heavy, etc.) more research is needed.

2.1 What Model Complexity Can We Afford?

- Consider a BTRee that indexes 100M records with a page size of 100. We can think of every b tree node as a way to partition the space, decreasing the error and narrowing the region to find the data

- We therefore say that the btree with a page size of 100 has a precision gain of 1/100 per node and we need to trabel a total of log_100N nodes. So the first node partitions the space from 100M to 1M, second from 1M to 10k and so on.

- Traversing a single b tree page with binary search takes roughly 50 cycles and is notoriously hard to paralellize

- In contrast, a modern PCU can do 8-16 SIMD operations per cycle, thus a model will be faster as long as it has a better precision gain than 1/100 per 50 * 8 = 400 arithmetic operations

- Note tihe calc still assumes that all B Tree pages are in the cache, a single cache miss costs 50-100 additional cycles and would thus allow for even more complex models.

 -Additionally, machine learning accelerattrs are entirely changing the game
- They allow much more complex models in the same amount of time and offload compute from the cpu

- For exAMPLE (from the paper idk what is now in 2026), NvIDIA's latest Tesla V100 GPU is able to achieve 120 teraflops of low precision deep learning arithmetic operations (~60k ops per cycle)

- Assuming that the entire learned index fits into the gpus memory, in just 30 cycles we could execute 1 million neural net operations.

2.2 Range Index Models are CDF Models
- An interesting observation: a model that predicts the position given a key inside a sorted array effectively approximates the umulative distribution function. We can model the CDF of the data to predict the position as: p = F(KEY) * N where p is the position estimate, F(Key) is the estimated cumulative distribution function for the data to estimate the likelihood to observe a key smaller or equal to the lookup key, P(X <= Key), and N is the total number of keys.

- This implies that indexing literally requires learning a data distribution. A BTree "learns" the data distribution by building a regression tree. A linear regression model would learn the data distribution by minimizing the squared error of a linear funciton. Second, estimating the distribution for a dataset is a well known problem and learned indexes can benefit for decades of research. Third, learning the CDF plays also a key role in optimizing other types of index structures and potential algorithms as we will outline later in this paper. Fourth, there is a long history of research on how closely theoretical CDFs approximate empirical CDFs , that gives a foothold to theoretically understand the benefits of this approach.

2.3 A First, Naiive Learned Index

- Used 200M web-server log records with the goal of building a secondary index over the timestamps using TensorFlow. Trained a two layer fully connected neural network with 32 neurons per layer using ReLU activation functions. Afterwards, measured the lookup time for a randomly selected key with Tensorflow and Python as the frontend
- Around 1250 predictions per second. IE ~80k ns to execute the model with tensorflow. As comparison, a BTRee trabersal over the same data takes ~300ns and binary search over the entire data roughly ~900 ns. 

Naiive approach is limited in a few ways:
1. Tensorflow was designed to efficiently run larger models, not small models, and thus has significaiton invocation overhead, especially with Pyhton as front end
2. B Trees, or decision trees in general are really good in overfitting the data with a few operations as the recursively divide the space using simple if statments. In contrast, other models can be significantly more efficient to approximate the general shape of a CDF, but have probems being accurate at the individual data instanc e level. Thus, models like neural nets, poly reg, etc. might be more CPU and space efficent to narrow down the position for an item from the entire dataset toa region of thousands, but a single neural  net usually requires significantly more space and CPU time for the last mile, to reduce the rror further down from thousands to hundreds. (maybe interesting research here if none already)

- BTrees are extremely cache and operation efficient as they keep the top nodes always in cache and access other pages if needed. In contrast. standard neural nets requires all weights to compute a prediction, which has a high cost in the number of multiplications

3 The RM Index
- Learning Index Framework (LIF)
- Recursive Model indexes (RMI)
- Standard error based search strategies
- Primarily focus on simple, fully connected neural nets because of their simplicity and flexibility, but believe other types of modesl may provide additional benefits (maybe research here)

3.1 The Learning Index Framework

- Can be regarded as an index synthesis sytem; given an index specificaiton, LIF generates different index configs, optimizes them, and tests them automatically

- LIF trains models with TensorFlow when needed, but compiles each finished model down into lean, dependency-free C++ for the actual lookups, which is what makes learned indexes fast enough to beat a B-Tree. 80kns -> 30ns. (Research angle: this is a research prototype, not production grade because it's built to quickly try out lots of different configurations, it carries some extra baggae of its own, no hand tuned SIMD.)

3.2 The Recursive Model index

- The core idea is solving the problem: one model alone can't get acurate enoguh

- RMI is a heierachy of small "expert" models where each one's prediction directly selects the next expert, turning the impossibly hard "find the exact position in 100M records" problem into a chain of easy narrow-it-down steps with nos searching in between which is what makes it accurate and fast

3.3 Hybrid Indexes

- You dont have to use the same kind of model everywhere in the hierarchy, mix and match and pick the right tool for each layer.

- Top layer needs to capture big, complex, overall shape of the data, so a small neural net (ReLU) is a good fit
- Bottom layers each handle a tiny, simple slice so you don tneed anything facny. Use thousands of cheap linear regressions, they're tiny in memory and fast to run
- If some chunk of data is too weird to learn, fall back to a BTree

- Hybrid indexes bound the worst case to a BTree's performance. Because of the automatic fallback, if the data is so pathological that nothing learns it, every single model gets swapped for a btree

- Hybrid indexes let each layer use th ebest model type for its job

3.4 Search Strategies and Monotonicity

- The model gets you close, and now you have to actually find the exact key (this is the "last mile" search)

- A range index offers upper_bound(key) and lower_bound(key). 

- The Setup: normally, plain search wins, a known result in this field is that for finding a key in a sorted array, fancy search algorithms usually arent worth it - plain binary search beats them because of less overhead

- Learned indexes have an edge, a BTree only tells you which page the key is in, theny ou search the hwole page. A learned model predicts the actual postiion, a specific spot, not just a neighborhood

Model biased Search: just binary search, but it starts at where the model predicted the position instead

- Biased Quaternary Search: A variant that checks three points at once instead of one. This is because modern hardware can pre fetch several memory locations simultaneously, if the data isn't already in cache, grabbing three spots in parallel hides memory latency better than grabbing one, waiting, then grabbing the next. The clever part is where they put the three points at, right around the prediction spread out by some sigma (a measure of typical error). They're betting the answer is near pos, so thye cluster their first three probes tightly around itbefore falling back to normal quaternary search if needed.

- Both strats use the model's prediction to start searching in the right place

- For all these, they need a bounded region to search in, they get it form th emin/max error stored per bottom-stage bodel - run every key through, record the worst over and under shoot, and that defines the ansewr is somehwere in here.

- missing keys need monotinicity

3.5 Indexing Strings

- Everything so far assumed keys are numbers, but databases index strings too - names, urls, ids, and a model cant do math on strings.

- Step 1: Tokenization
- Step 2: Fixed length
- Step 3: Same architecture, bigger input.
- Step 4: We're just scratching the surface (future work and possible research angle)
    - Smarter tokenization - instead of one numbr per character, borrow techniques from natural lagnaueg processing like wordpieces, which break strings into meaningful chunks 
    - Suffix trees + learned models
    - Fancier architecutres - RNNs and CNNs which are purpose built for sequences like text and might model strings more effectively

- Short version: to index strings you turn each one into a fixed length vector of character values (ASCII numbers, truncated or zero padded to length N), then feed that into the same RMI neural net hierarchy sa before. The catch is that work now scales with string length N which is why stirng indexing is costlier and wins over B-Trees are smaller - and they flag smarter tokenizaiton (like wordpieces) and sequence models as the obvious next step

3.6 Trainig

- Building a learned index is fast. The cheap linear models train in a single pass via  a direct formula, and the small neural nets converge in just a handful of passes. A 200M record index trains in seconds, helped by the fact that the top model can stop early since it only needs rough accuracy.

- This addresses static, read only build time. It says nothing about updating the index when new data arrives - which is hard

3.7 Results

- On integers, learned indexes beat even a tuned production B-Tree. Up to 3x faster and ~100x smaller, and beat the other specialized baselines on the combined size/speed front. On strings, the wins shrinkm because model execuiton and string comparisons are both pricier, which is exactly where hybrid models and msarter search strategies start to earn their keep

- Learned indexes win biggest when the data is learnable and comparisons are cheap, and the gap narrows when either of those gets harder - which is consistent, sinsible pattern rather than a magic bullet claim

4 Point index and the Hash model Index

- Now switching from range indexes to point indexes

- BTrees find ranges, a hash map answers: is key x here and where?

- Core problem with hash maps: Conflicts.

- Even with perfectly random hash function and a table with exactly as many slots as records, (100M slots, 100M keys) you still get ~33% conflicts, about 33M slots collide.

- Every conflict forces the hash map to do extra work

- The key insight: What causes conflicts? The hashing function spreading keys unevenly. A learned model that knows the data distribution can spread them more evenly than blind randomness, specifically h(K) = F(K) * M where F(K) is the learned CDF (same CDF model from the range index sections) and M is the table size. 

- Important caveats:
    - only helps if the data is non uniform
- the payoff depends on the hash map architecture. learned hashing is orthogonal - it plugs into any hash map type, but whether the model's execution cost is worth it depends on how expensive conflicts are in your setup.

Result

Map Data: 35.3% -> 7.9% conflicts
Web data: 35.3% -> 24.7% reduction
Lognormal: 35.4% -> 25.9%

- Pattern: Map data gets the biggest win because the model learns its CDF accurately, the messier the datasets get smaller smaller

- Larger payloads / separate chaining: big win.
- Tiny payloads: not worth it
- Distributed settings: This is where it shines. In something like NAM DB a conflict means an extra network round trip costing microseconds (orders of magnitutde more than nanoseconds)

- In all: the same CDF model that predicted positions can be repurposed as a hash function — F(key) × M — that spreads keys evenly and cuts conflicts by up to 77%. Whether that's worth the ~25-40ns model cost depends on how expensive your conflicts are: not worth it for tiny in-memory payloads (Cuckoo wins), clearly worth it for large payloads, and a potentially massive win in distributed systems where each conflict costs a network round-trip.

Everything reduces to learning the CDF. Range index = CDF predicts position. Point index = CDF spreads key to avoid conflicts.

5 Existence Index (Bloom Filters)

- Bloom filters are space efficient but still hungry at scale, 1 billion records ~1.76 GB and pushing the FPR downt o 0.01% needs ~2.23 GB. That's a lot of RAM needed

- The insight: A traditional Bloom filter treats keys as random — it learns nothing about what makes a key a key. But often there is structure separating keys from non-keys. Their extreme example: if your keys are exactly the integers 0 to n, you don't need a Bloom filter at all — just check 0 ≤ x < n, constant time, near-zero memory. Real data isn't that clean, but if a model can learn the boundary between keys and non-keys, it can shrink the filter.

- Point index hash wants few collision among keys, bloom filter wants lots of collisions among keys, lots among non keys, but few between the two

Approach 1: Bloom Filter as a Classification Problem
- train a model that outputs the probability that x is a key
- you train it on both keys and known non keys, thats why you need a non key dataset. for strings (like urls) theyd use an RNN/CNN. output is probability via sigmoid; trained with log loss

- now pick a threshold. if f(x) >= T, call it a key; otherwise not

- but a model isnt perfect - itll have both false positives AND false negatives. false positives are fine, but false negatives are forbidden

- the fix: the overflow bloom filter. take all the real keys that model wrongly rejects and build a small traditional bloom filter containing just those. now the full thing is:
1. ask the model if f(x) >= T -> key exists
2. else, don't trust it: check the overflow bloom filter as a backup

- this is model + auxiliary structure decomposition

- a bloom filter's size scales with how many keys it must hold. the overflow filter only holds the model's false negatives

approach 2: model as a hash function. 

- use the same models output as a hash function feeding a bloom filter (d = f(x) * m). because the model was trained to separate keys from non keys it naturally maps most keys to the high end of the bit array and non keys to the low end

5.2 Results
- 1.7M blacklisted phishing URLs. non keys are  amix of random and whitelisted URLs. They train a small character level RNN to tell phishing from non phising

A normal Bloom filter at 1% FPR: 2.04MB. The GRU model: just 0.0259MB — tiny.
At 0.5% FPR, the model has a 55% false-negative rate, so the overflow filter must hold 55% of keys → total 1.31MB, a 36% reduction.
At a stricter 0.1% FPR: FNR rises to 76%, total drops from 3.06MB to 2.59MB, a 15% reduction.

- a learned Bloom filter trains a model to predict "key vs. non-key," then pairs it with a small overflow Bloom filter that holds only the model's false negatives — restoring the sacred no-false-negatives guarantee while shrinking total memory (36% on phishing URLs at 0.5% FPR). The savings depend on model accuracy and shrink as you demand lower FPRs, but the model can draw on richer features than a plain filter ever could.












