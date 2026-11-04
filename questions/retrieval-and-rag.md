# Retrieval and RAG

The useful thing about this topic in 2026 is that several widely-repeated beliefs don't survive contact with controlled benchmarks. Being able to say *"actually, the evidence is more awkward than that"* — with a citation — is worth more than fluency in any particular framework.

---

### 1. When does BM25 beat an agentic retrieval loop?

`verified 2-1`

<details>
<summary>Answer</summary>

At scale, and the crossover is measurable.

*BM25 Wins at Scale: A Scaling Study of Retrieval-Augmented Generation Paradigms* (arXiv 2607.26497, July 2026, USTC / Metastone / BAAFS) ran **28 strictly nested corpus tiers** growing 1.25x per rung — from 1,144 docs / 1.7M tokens to 511,959 docs / 600.8M tokens, roughly 450x — holding questions, reader model (Qwen3.6-27B at temperature 0) and judging protocol fixed.

- BM25 **overtakes** an agentic File-System Agent paradigm at around **10 million corpus tokens** (crossover interval 8.5M–10.6M)
- It leads at **every larger tier**, with the gap approaching **20 points (50.5 vs 30.7)** at the full 601M-token corpus
- The agent consumed roughly **39x more query tokens** at the smallest tier
- Dense embedding retrieval was efficient but *less accurate than BM25*, losing ground as the corpus grew

**State it as a crossover, not a winner** — the paper does. The agent leads *below* ~10M tokens and on completeness and conflict-type questions. The authors' own framing is the line to remember:

> agentic reasoning works best after ranked discovery rather than in place of it

**Why this is graded 2-1:** one benchmark, one reader model, unrefereed preprint. Say "one controlled study found", not "it's established".

</details>

---

### 2. Is GraphRAG worth the indexing cost?

`single-source`

<details>
<summary>Answer</summary>

Usually not, if you're justifying it on QA accuracy alone.

**GraphRAG-Bench** evaluated nine methods (RAPTOR, LightRAG, GraphRAG, G-Retriever, HippoRAG, GFM-RAG, DALK, KGP, ToG) across 20 textbooks and 16 disciplines, scoring graph construction, retrieval and generation separately:

- Some methods **actively degrade** accuracy versus baselines (DALK, G-Retriever); several give only marginal gains (LightRAG, ToG, KGP)
- Best method (RAPTOR) reached **~73.58%** average accuracy versus **~71.66%** for BM25, **~71.71%** for TF-IDF, and **~70.68%** un-augmented — **under two points over cheap lexical retrieval**
- Construction costs span **10.1M tokens** (RAPTOR) to **83.9M tokens** (LightRAG), build times ~4,674–20,396 seconds, per-query retrieval from **0.02s to 89.38s**
- **Every** evaluated method *reduced* accuracy on mathematics questions; ethics was also weak — attributed to symbolic/procedural reasoning that graph retrieval doesn't help with

**The methodological catch that matters most:** the baselines beaten are *sparse lexical retrievers*. This is not evidence that GraphRAG beats a modern hybrid dense + BM25 + reranker pipeline — which is the comparison that actually decides production architecture.

A separate April 2026 benchmark (**RAGSearch**) found that adding **agentic multi-round retrieval to plain dense RAG closes most of the gap to GraphRAG**, largest gains when the search policy is RL-trained. An agent loop is a partial substitute for building the graph at all.

</details>

---

### 3. So when *would* you build a knowledge graph?

<details>
<summary>Answer</summary>

When the value is **identity and relationships**, not retrieval scoring.

Graphs earn their place with: entity-resolution problems, multi-hop relationship queries, and ontologies encoding real domain constraints that you want to *validate* against rather than merely embed.

Concrete shapes that justify a graph:

- **The natural key is broken.** An identifier that's only unique within a partition, or reused across sources, so records must be resolved rather than joined.
- **Questions are multi-hop over relationships**, not lookups over text — "which suppliers appear under more than one legal entity", not "what does clause 4 say".
- **An ontology encodes real constraints** you want to *validate* against rather than merely embed — SHACL shapes failing loudly on malformed data is a capability embeddings simply don't have.

Justifying a graph on identity and ontology grounds is a far stronger answer than quoting a benchmark delta — especially given the benchmark deltas are ~2 points.

**The four-axis framing to use when asked "should we use GraphRAG?"**: report answer accuracy, offline preprocessing cost, online inference efficiency, and stability. Most published GraphRAG-vs-RAG comparisons are unreliable precisely because they vary the LLM backbone, retrieval budget and inference protocol simultaneously, and subsample test sets. Saying that out loud is the senior signal.

</details>

---

### 4. How do you evaluate a RAG system?

`single-source`

<details>
<summary>Answer</summary>

In **two layers**, because there is no single end-to-end score that substitutes for both.

- **Retrieval** gets classical IR metrics — Recall@k, Precision@k, MRR. This is where most real failures live, and it's cheap to measure with no LLM in the loop.
- **Generation** gets error analysis plus a **statistically validated** LLM judge.

Then say the unpopular part: **off-the-shelf framework metrics are a trap.** Husain and Shankar argue generic metrics waste time and create false confidence, and that similarity metrics like ROUGE and BERTScore aren't useful for evaluating LLM outputs in most applications (with a stated exception for search and recommendation).

Build application-specific evals derived from failure modes you actually observed in traces. A `faithfulness: 0.87` in a README invites exactly the question you don't want: *what does that number mean, and how do you know?*

See [`reference/evaluation-playbook.md`](../reference/evaluation-playbook.md) for the judge validation protocol.

</details>

---

### 5. How would you chunk a corpus that has its own published structure?

<details>
<summary>Answer</summary>

**Don't chunk by token window.** If the source publishes structure — sections, articles, clauses, headings — that structure is usually more meaningful than anything a splitter will infer, and it's often *authoritative* (in legal or regulatory text, the unit boundary carries legal weight).

Chunk at the published unit. Every chunk is then citable with a stable identifier rather than being an arbitrary character span, which is what makes grounded citation possible at all.

Practical notes that generalise:

- **Read the source format before choosing a strategy.** Structured corpora frequently expose an index endpoint plus per-unit retrieval, and content negotiation is often narrower than the docs imply — XML-only endpoints returning HTTP 400 on `Accept: application/json` are common in government and standards data.
- **Carry versioning metadata onto every chunk.** For any corpus that gets amended, an answer needs to cite *which version* it relied on, or citations silently drift as the source updates.
- **Splitting with overlap is a fallback** for units that exceed a token threshold — not the default.

The principle worth stating: **when a document format carries real structure, using it beats any chunking heuristic.** This is a good decision to lead with, because it demonstrates reading the source rather than reaching for a default splitter.

</details>

---

### 6. Design RAG for 10 million documents, end to end.

<details>
<summary>Answer</summary>

Start by asking what the documents are and who can see them, because those two answers drive more of the design than the vector database does. Then walk the pipeline in order and put numbers on the parts that cost money.

**Ingestion.** Parsing is where quality is won or lost at this size. PDFs with tables, scanned pages and slide decks each need their own parser, and you want a per-format sample checked by a human before you embed 10M of anything. Store the parsed text with a content hash so re-ingestion only re-embeds what changed.

**Chunking.** Use the document's own structure where it has one (see [question 5](#5-how-would-you-chunk-a-corpus-that-has-its-own-published-structure)), and fall back to a few hundred tokens with overlap where it doesn't. Attach document ID, section path, version, timestamp and the ACL to every chunk. Assume about 10 chunks per document, so roughly 100M chunks.

**Index choice and memory.** This is the maths interviewers want to see:

- 100M vectors at 1,024 dimensions in float32 is 100M x 1,024 x 4 bytes, about 410 GB of raw vectors.
- HNSW adds graph links on top. With M=32 you store around 2M neighbour IDs on layer 0, so about 64 x 4 = 256 bytes per vector, another 25 GB or so.
- int8 scalar quantization cuts the vectors to about 100 GB. Product quantization at 64 bytes per vector gets you to about 6.4 GB, with a recall cost you then recover by rescoring the top few hundred against full-precision vectors kept on disk.

So the realistic options are HNSW with quantized vectors sharded across a few large-memory nodes, or an IVF-PQ or disk-based graph index (DiskANN-style) that keeps compressed codes in RAM and full vectors on SSD. Smaller embedding dimensions (Matryoshka-style truncation to 256 or 512) are often the cheapest lever of all, and you should measure recall on your own data before and after each compression step.

**Hybrid retrieval.** Run BM25 alongside the dense index and fuse the lists (see [question 8](#8-bm25-dense-or-hybrid-when-do-you-use-each)). At this scale lexical retrieval is cheap, strong, and catches identifiers that embeddings miss.

**Reranking.** Take the top 50 to 200 fused candidates through a cross-encoder and keep the best 5 to 20 for the prompt. Prove it helps before you pay for it (see [question 9](#9-you-added-a-reranker-how-do-you-prove-it-helped)).

**Permissions.** Filter at retrieval time inside the index, using ACL metadata on each chunk, and never by asking the model to ignore what it shouldn't see. Post-filtering after top-k is the common mistake: a user with narrow access gets a top-k that is mostly filtered out and sees empty or poor results. Prefer pre-filtered or filtered-HNSW search, or partition the index by tenant when tenants are large. ACL changes must propagate quickly, so treat them as their own update stream.

**Freshness.** Use change data capture or a crawl queue, re-embed changed chunks by hash, and handle deletes as tombstones that are actually purged (deleted content showing up in answers is a compliance problem). Decide up front whether a model upgrade means re-embedding all 100M chunks, since that is a batch job measured in days and a bill you should estimate before you choose the model.

**Context assembly and cost.** How you pack chunks into the prompt and keep it cache-friendly is covered in [context engineering](context-engineering.md#2-long-context-models-advertise-1m-tokens-why-not-just-put-everything-in) and [cost and caching](cost-and-caching.md#3-what-makes-a-prompt-cache-friendly-and-how-do-people-accidentally-defeat-their-own-cache).

**Eval.** Two layers, as in [question 4](#4-how-do-you-evaluate-a-rag-system): recall@k on a labelled query set for retrieval, and failure-mode-driven checks on the answers. Build the labelled set before the index so you can make every choice above (chunk size, quantization, hybrid weights) on evidence.

The trade-off to say out loud: most of the risk sits in parsing, permissions and freshness. The index is the part with the best off-the-shelf answers.

</details>

---

### 7. Your RAG system gives wrong answers. How do you debug it?

`single-source`

<details>
<summary>Answer</summary>

First decide whether retrieval or generation failed, because the fixes have nothing in common. Take a labelled set of failing questions, mark the chunk that contains the answer, and check whether that chunk was in the retrieved top-k.

- **Right chunk not retrieved.** It's a retrieval problem. Measure recall@k across the labelled set to see how widespread it is.
- **Right chunk retrieved, answer still wrong.** It's a generation problem. Check faithfulness: does the answer follow from the context, or did the model ignore it, misread it, or fill a gap from its own weights?
- **No chunk contains the answer.** It's a corpus problem, and the system should have said "I don't know".

Barnett et al., *Seven Failure Points When Engineering a Retrieval Augmented Generation System* (arXiv 2401.05856, 2024), list failures drawn from three case studies: missing content, missed top-ranked documents, the answer not making it into the context, the answer in context but not extracted, wrong format, wrong specificity, and incomplete answers. It's a useful checklist, though it comes from experience reports and not a controlled study.

I work the taxonomy roughly in pipeline order, because upstream bugs masquerade as downstream ones:

1. **Ingestion and parsing.** Open the stored text for a failing document. Tables flattened into nonsense, missing pages from scanned PDFs and boilerplate drowning the content are common and invisible from the answer alone.
2. **Chunking.** The answer may be split across two chunks, or separated from the heading that gives it meaning. Fixes are structure-aware chunking or attaching the section path to each chunk.
3. **Query mismatch.** Users ask in different words than the documents use, or use abbreviations and IDs that the embedding model handles badly. Look at the query next to the gold chunk. Hybrid retrieval, query rewriting or a domain-tuned embedder help here.
4. **Ranking.** The gold chunk is at rank 40 and you pass the top 10. Recall@50 versus recall@10 tells you whether a reranker or a larger k would fix it.
5. **Context ordering and assembly.** The chunk made the prompt but sits in the middle of a long context, or was truncated by a token budget. Models use information in the middle of long contexts less reliably (Liu et al., "Lost in the Middle", arXiv 2307.03172), so put the strongest evidence first and pass fewer, better chunks.
6. **Generation.** The context is good and the model still errs. Tighten the instructions on citing and abstaining, check for conflicting chunks (old and new versions of a policy), and evaluate with a validated judge as in [question 4](#4-how-do-you-evaluate-a-rag-system).

The judgment call: resist tuning the prompt until you know retrieval works, since prompt fixes for retrieval failures overfit to the examples you looked at. Keep the failing questions as a regression set so the fix stays fixed.

</details>

---

### 8. BM25, dense, or hybrid: when do you use each?

`single-source`

<details>
<summary>Answer</summary>

Default to hybrid, and know the cases where one side alone is enough. [Question 1](#1-when-does-bm25-beat-an-agentic-retrieval-loop) covers BM25 against agentic search at scale; this question is about the lexical versus dense choice inside a single retriever.

**BM25 alone** makes sense when queries carry exact tokens: error codes, part numbers, function names, legal citations, people's names. Dense embeddings blur these, because "ERR-4012" and "ERR-4021" look nearly identical to an embedding model. BM25 also needs no training, no GPU, and is easy to debug since you can see why a document matched.

**Dense alone** makes sense when queries and documents use different vocabulary for the same idea (a user's "my card got eaten" against a page titled "ATM retained card"), and when you have in-domain training data or a strong embedder tested on your corpus.

**Domain shift** is the argument against trusting dense retrieval blindly. BEIR (Thakur et al., arXiv 2104.08663, 2021; NeurIPS 2021 Datasets and Benchmarks) tested retrievers zero-shot across 18 datasets and found BM25 a strong baseline that many dense models trained on MS MARCO failed to beat out of domain. Embedders have improved a lot since, but the lesson holds: an embedder's leaderboard score is a prior, and your own labelled queries are the test.

**Hybrid** runs both and fuses the ranked lists. Reciprocal Rank Fusion (Cormack, Clarke and Büttcher, SIGIR 2009) scores each document as the sum of 1/(k + rank) across the lists, with k = 60 in the paper. It uses ranks only, so you don't need to calibrate BM25 scores against cosine similarities, and in the paper it beat the best individual input system as well as Condorcet fusion. Weighted score fusion can beat RRF once you have labelled data to tune the weights, at the cost of retuning whenever an embedder or the corpus changes.

What I'd say in the interview: hybrid costs a second index and some latency, and in exchange you stop losing the identifier queries and the paraphrase queries. If measurement on your own queries shows one side contributes nothing, drop it.

</details>

---

### 9. You added a reranker. How do you prove it helped?

<details>
<summary>Answer</summary>

Measure it in three places: offline retrieval metrics, offline answer quality, and an online test. A reranker that improves nDCG and leaves answers unchanged hasn't paid for its latency.

**What it is.** A bi-encoder embeds the query and documents separately, so document vectors are precomputed and search is fast. A cross-encoder reads the query and one candidate together and outputs a relevance score. It's more accurate because it sees the interaction between the two, and much slower because nothing can be precomputed: scoring 100 candidates means 100 forward passes (batched) per query. That's why you use it as a second stage on a short list.

**Latency cost.** Budget it explicitly. Reranking 100 candidates with a small cross-encoder on a GPU is typically tens of milliseconds; a large model or a hosted API call adds more plus network time. Candidate depth is the dial: more candidates give the reranker more to work with and cost linearly more.

**Offline retrieval metrics.** On a labelled query set, compare nDCG@k and recall@k for the final k you pass to the model, with and without the reranker. Also report recall at the candidate depth (say recall@100 of first-stage retrieval). That number is the ceiling: a reranker can only reorder what retrieval returned, so if the gold chunk isn't in the top 100, no reranker will save it, and the work belongs in first-stage retrieval.

**End-to-end answer eval.** Run the same question set through the full pipeline both ways and grade answers with your validated judge or human labels (see [question 4](#4-how-do-you-evaluate-a-rag-system)). Better ranking may not change answers if the model was already finding the gold chunk at rank 8 in a 10-chunk context. It often matters most when it lets you pass fewer chunks, which also saves tokens.

**Online A/B test.** Offline sets drift from real traffic, so ship behind a flag and compare on metrics you defined beforehand: answer acceptance or thumbs-up rate, follow-up rephrasing rate, escalation to a human, citation clicks, and p95 latency as a guardrail. Fix the sample size before starting, and don't stop early when the line looks good.

If you use a hosted reranker, check the vendor's current documentation for model names, input limits and pricing at the time you build, since these change often. The judgment to show: if recall at the candidate depth is low, fix retrieval first; a reranker is the second thing you tune.

</details>

---

### 10. How do you choose chunk size and overlap?

`single-source`

<details>
<summary>Answer</summary>

I run it as an experiment against my own eval set, because no universal value exists. Published results change with the corpus, embedder and retriever, so they tell me what to sweep and my eval tells me what to ship. If the corpus has published structure, I start from that unit as in [question 5](#5-how-would-you-chunk-a-corpus-that-has-its-own-published-structure) and sweep only for units that are too long.

**What constrains size.** The embedding model's input limit is the ceiling, and you need to know what happens above it. Voyage truncates over-length text by default unless you set `truncation=false`, so a chunk can lose its tail with no error. Below that, short factual queries tend to suit small chunks and spread-out answers suit large ones, document structure suggests boundaries, and chunk size times k has to fit the context budget.

**The trade-off.** Small chunks match precisely and carry little surrounding text. Chroma (Smith and Troynikov, 2024) measured token-level results with 5 retrieved chunks and `text-embedding-3-large`: a recursive splitter at 800 tokens with 400 overlap got recall 85.4 and precision 1.5, and at 200 tokens with no overlap got recall 88.1 and precision 7.0. The dataset is 472 LLM-generated queries over 328,208 tokens, which the authors call small. Bhat et al. (arXiv 2505.21700) found the best fixed size varied by dataset and embedder across 64 to 1,024 tokens; on COVID-QA, Stella peaked at 64 and Snowflake at 1,024.

**Separate the retrieval unit from the unit the model reads.** Embed small pieces and hand over the larger parent, which loosens the trade-off. LlamaIndex's auto-merging retriever embeds only leaf nodes and swaps in a parent when enough of its children are retrieved. You pay in storage and one more threshold, and I have no controlled study of the benefit to quote, so it goes in the sweep too.

**Overlap evidence conflicts,** so it gets its own arm.

- Bennani and Moslonka (arXiv 2601.14123) found no measurable gain on Natural Questions with SPLADE and no reranker (|ΔBERTScore| ≤ 0.004), while 20% overlap means 1.25x the chunks. Their grid lists overlaps of 0% and 20%, though their results text says 10 to 20%, so I quote the 20% figure. They ask for validation on specialized domains.
- Günther et al. saw no clear advantage from 16 tokens on 256-token chunks, and Qu et al. no notable gain from one overlapping sentence. Both overlaps are small.
- Chroma's result depends on the splitter. With `all-MiniLM-L6-v2` at 250 tokens, 125 overlap moved a token splitter's recall from 77.1 to 82.4 and a recursive one's from 78.5 to 78.7.
- El Bachyr et al. (arXiv 2604.12047) point the other way. On financial PDFs, going from no overlap to 25% lifted page-level MRR from 0.529 to 0.658 on FinanceBench and from 0.735 to 0.833 on TableQuest, with 50% slightly lower.

**Restoring lost context directly.** Anthropic's Contextual Retrieval prepends 50 to 100 tokens of LLM-written context to each chunk. In their own experiments, top-20 failure fell from 5.7% to 3.7% with contextual embeddings, 2.9% adding contextual BM25, and 1.9% adding reranking. They estimate $1.02 per million document tokens, assuming prompt caching and Claude 3 Haiku. Late chunking (Günther et al., arXiv 2409.04701) runs a long-context embedder over the whole text and pools per chunk afterwards, with no LLM. Across three models and four BeIR datasets, average nDCG@10 at 256-token chunks went from 52.2 to 54.0, unevenly, and naive chunking won on some reading-comprehension tasks with large chunks. One caveat on that setup: the paper says its main runs use no overlap, but its Table 2 figures for jina-embeddings-v2-small match the with-overlap column of its own overlap appendix (Table 6), so the exact configuration behind the averages is unclear. These authors built some of the models tested, and their comparison with contextual embeddings covers one fictional document. Semantic chunking has to earn its cost: Qu et al. (arXiv 2410.13070) saw it win mainly on artificially stitched documents, with fixed-size often better on real ones.

**Running the sweep.** Compare at a fixed token budget, because smaller chunks at the same k return less context. By my arithmetic, Chroma's 5 chunks are about 685 tokens at the 137-token mean and about 3,300 at the 661-token mean. Bennani and Moslonka fill a token budget in rank order for this reason. Remove duplicated overlap text before counting. Measure recall and token-level precision at that budget, then grade answers with the two layers from [question 4](#4-how-do-you-evaluate-a-rag-system), keeping your real reranker in the pipeline. Retrieval metrics can't show what context does to the model: in Bennani and Moslonka's setup, BERTScore held from 0.5k to 2.5k tokens and fell about 4 to 5% relative by 10k, a point they say to re-tune per model. My own default to test, not a finding: sentence-aware splits at 300 tokens with no overlap, against 150, 600 and a 20% overlap arm.

**Re-indexing and document classes.** Every arm means new embeddings, and content hashes don't help because the chunks themselves change. I sweep on a sample, confirm the winner on the full corpus since a sample has fewer distractors, and version the chunking config so a later change is a batch job with a known bill. Break results down by document class too, since contracts, tickets and tables may want different settings. Each extra config is more to maintain and re-test, so I split only where a class clearly differs beyond the noise in paired confidence intervals.

Sources: Chroma, "Evaluating Chunking Strategies for Retrieval" (trychroma.com/research/evaluating-chunking); Anthropic, "Introducing Contextual Retrieval" (anthropic.com/news/contextual-retrieval); arXiv 2409.04701, 2410.13070, 2505.21700, 2601.14123, 2604.12047; Voyage embeddings API reference; LlamaIndex auto-merging retriever docs. All fetched 2026-11-04.

</details>
