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
