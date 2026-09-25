# Uncertainty quantification

**A third evaluation layer, distinct from the other two.** [evaluation.md](evaluation.md) asks *did the system do well, offline, over a test set*. [model-evaluation.md](model-evaluation.md) asks *how good is this model at modelling language*. This file asks the runtime question: **given this one response, right now, should I trust it?**

That question is what hallucination detection, selective answering and "refuse rather than guess" all rest on — and it is a common interview topic precisely because most candidates have only ever used an LLM's stated confidence, which is worthless.

Reference implementation throughout: **[uqlm](https://github.com/cvs-health/uqlm)** (CVS Health, Apache 2.0) `primary`, fetched 2026-09-25.

---

### 1. Given a single response, how do you decide whether to trust it?

`foundational`

<details>
<summary>Answer</summary>

There are three families, and the right answer names the trade-off between them rather than picking one.

**Black-box (consistency-based).** Sample the same prompt *N* times and measure how much the answers agree. If the model knows, it converges; if it's confabulating, it wanders. Measures include non-contradiction probability, entailment probability, number of semantic sets, exact match, BERTScore and cosine similarity.
*Cost: N× generations. Latency: medium to high. Works on any model, including one behind an API that returns nothing but text.*

**White-box (token-probability-based).** Use the logprobs the model already produced. Single-generation measures include sequence probability, length-normalised sequence probability, minimum token probability, probability margin and top-k negentropy.
*Cost: effectively zero, no extra calls. Requires logprob access — which many hosted APIs do not give you.*

**LLM-as-judge.** Ask a model to assess the response — categorical, continuous, Likert, or a panel of judges.
*Cost: extra calls. Carries every judge bias in [evaluation.md](evaluation.md), so it needs the same validation.*

And **ensembles** that combine the above by weighted averaging (BSDetector, supervised ensembles) — usually the strongest, and the most work.

The senior instinct: **pick the family your deployment constraints allow, then validate it.** If you can't get logprobs, white-box is off the table no matter how cheap it is.

</details>

---

### 2. What is semantic entropy, and why is it better than token-level entropy?

`foundational`

<details>
<summary>Answer</summary>

The problem it solves: **the same meaning has many surface forms.** Token-level entropy over generations treats "Paris", "It's Paris", and "The capital is Paris" as three different answers and reports high uncertainty — when the model is in fact completely certain. Linguistic variation gets misread as semantic uncertainty.

**Semantic entropy fixes this by computing entropy over meanings rather than token sequences:**

1. Sample *N* generations for the prompt
2. Cluster them into semantic equivalence classes, typically using **bidirectional entailment** — two answers are the same meaning if each entails the other, judged by an NLI model
3. Compute entropy over the *cluster* probabilities, not the sequence probabilities

Low semantic entropy means the model keeps saying the same thing in different words: confident. High semantic entropy means it keeps saying genuinely different things: confabulating.

Introduced in **Kuhn, Gal & Farquhar, *Semantic Uncertainty: Linguistic Invariances for Uncertainty Estimation in Natural Language Generation***, ICLR 2023 (arXiv 2302.09664), with the follow-up **Farquhar, Kossen, Kuhn & Gal, *Detecting hallucinations in large language models using semantic entropy***, Nature 2024.

Variants you may see: *discrete* semantic entropy (cluster counts, no probabilities needed — so it works black-box), and *semantic density*. uqlm implements both discrete and probability-weighted forms.

The one-line version worth having ready: **entropy over meaning classes, not over token strings.**

</details>

---

### 3. Why is raw sequence probability a poor confidence score?

`foundational`

<details>
<summary>Answer</summary>

Two independent problems.

**Length bias.** Sequence probability is a product of per-token probabilities, so it decays geometrically with length. A ten-token answer is near-certainly "more probable" than a fifty-token answer regardless of which is correct. That's why **length-normalised sequence probability** exists — the geometric mean per token rather than the product. Always normalise, or you're ranking by brevity.

**It measures fluency, not truth.** High token probability means the model finds the continuation *likely*, and a confidently-held false belief produces exactly that. Fluent, well-trodden, wrong text scores high. This is the deeper limitation and no amount of normalisation fixes it.

Which is why the sharper single-generation signals are things like **minimum token probability** (the weakest link — one very uncertain token can be where the fabrication entered) and **probability margin** (the gap between the top token and the runner-up, which captures whether the model was actually deciding between alternatives).

</details>

---

### 4. Your provider doesn't return logprobs. What can you still do?

`foundational`

<details>
<summary>Answer</summary>

Everything black-box, and nothing white-box. This is a genuinely common production constraint and worth having an answer ready for.

What remains available:

- **Sample N responses** at non-zero temperature and measure agreement
- **Non-contradiction probability / entailment probability** — run an NLI model over response pairs
- **Number of semantic sets** — cluster by meaning and count the clusters; more clusters means more uncertainty. This is discrete semantic entropy's cheaper cousin and needs no probabilities at all
- **Exact match**, or embedding cosine similarity, as crude fallbacks
- **Self-reflection / P(True)** — ask the model whether its own answer is correct. Cheap, but it inherits the model's miscalibration, so validate before trusting it

The cost is the honest part of the answer: **you're paying N× inference for every scored response.** In a latency-sensitive path that may be unaffordable, which pushes you toward scoring a sample of traffic rather than every request, or toward a cheaper model for the consistency samples.

</details>

---

### 5. Why is one confidence score for a long answer close to useless?

`foundational`

<details>
<summary>Answer</summary>

Because a paragraph is not one claim. A five-sentence answer routinely mixes four well-grounded statements with one invented date — and a single aggregate score either flags the whole answer (so you discard four good claims) or passes it (so the fabrication ships). Averaging destroys exactly the signal you need.

The answer is **claim-level uncertainty quantification**: decompose the response into atomic claims or sentences, score each independently, then decide per claim. uqlm exposes this as long-text scorers — LUQ, graph-based scorers, and generalised long-form semantic entropy.

This matters disproportionately for **RAG with citations**, which is the common case. Per-claim scoring lets you do the thing users actually want: keep the supported sentences, mark or drop the unsupported one, and attach the citation to the specific claim it supports rather than to the paragraph.

It also composes with the deterministic checks in [retrieval-and-rag.md](retrieval-and-rag.md) — verify mechanically that a cited source exists and was retrieved, *then* apply a UQ score to the claims that survive.

</details>

---

### 6. You have a confidence score. How do you know it's any good?

`foundational`

<details>
<summary>Answer</summary>

**A confidence score is an instrument, and an unvalidated instrument is decoration.** This is the same discipline as validating an LLM judge, and the connection is worth drawing explicitly.

**Calibration** is the property you want: among responses scored 0.8, roughly 80% should be correct. Measure it with a **reliability diagram** (confidence bucket vs observed accuracy) and summarise with **expected calibration error**. A score can rank perfectly and still be badly calibrated — useful for triage, useless as a probability.

**Discrimination** is the other half, and often what you actually care about: does the score separate correct from incorrect answers? Measure with **AUROC** over a labelled set. High AUROC with poor calibration is fine if you only need a threshold.

**The threshold is a product decision, not a statistical one.** Plot the **risk–coverage curve**: as you refuse more, how fast does error on what you *do* answer fall? That curve is how you choose between "answer 95% of questions with 8% error" and "answer 70% with 2% error" — and it makes selective answering a business conversation rather than a magic number.

Finally, the trap: **calibration is distribution-specific.** A threshold tuned on your eval set silently degrades when the input distribution shifts, so it needs monitoring in production, not one-time tuning.

> Honest note on tooling: uqlm's repository states no benchmark results, so treat it as a well-organised implementation of published methods, not as evidence that any one scorer wins. Which scorer works on *your* data is an empirical question you have to answer yourself.

</details>
