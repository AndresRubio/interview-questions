# Data curation

What goes into the weights, before anyone argues about LoRA rank or DPO versus PPO. [post-training.md](post-training.md) covers the training methods (question 3 there for SFT versus preference optimization, question 7 for distillation failure); this file is about the data those methods consume. Contamination *detection* lives in [model-evaluation.md question 7](model-evaluation.md#7-why-is-a-low-perplexity-on-a-public-benchmark-corpus-weak-evidence), and judge validation in [evaluation.md](evaluation.md); this file covers the curation side of both: keeping eval data out, and turning what the evals find into training data.

---

### 1. LIMA fine-tuned on 1,000 examples. Does that mean data quantity doesn't matter for SFT?

`single-source`

<details>
<summary>Answer</summary>

LIMA took a 65B LLaMa base and fine-tuned it with plain supervised loss on **1,000 carefully curated prompt/response pairs**, with no RL and no preference modelling. In the paper's human study, its responses were equivalent to or strictly preferred over GPT-4's in 43% of cases, 58% against Bard and 65% against DaVinci003 (Zhou et al., 2023, *LIMA: Less Is More for Alignment*, arXiv:2305.11206). The authors call their explanation the "superficial alignment hypothesis": almost all knowledge comes from pretraining, and SFT mostly teaches format and style.

AlpaGasus points the same way from the other direction. It filtered Alpaca's 52k instructions down to 9k using ChatGPT as a quality rater, and the filtered model significantly outperformed the original Alpaca in GPT-4-judged and human evaluations, while training 5.7x faster (Chen et al., 2023, arXiv:2307.08701).

What a senior answer adds is the scope of that result:

- **It holds for style and format, not for new capability.** If SFT is meant to add knowledge or skill the base model lacks, such as a domain's facts or a new tool protocol, 1,000 examples will not do it. LIMA's own claim is that the knowledge was already in the base model.
- **"Carefully curated" is the expensive part.** A thousand examples chosen for diversity and written to a consistent standard cost more human attention than 50k scraped ones. The saving is in compute, not in effort.
- **Diversity beats count once quality is fixed.** Ten near-identical examples of the same task add almost nothing, so deduplicate (question 2) before you count.

The instinct to show: when an SFT run underperforms, audit a random 100 examples by hand before you collect more. Bad labels at 5% hurt more than a missing 10x in volume.

</details>

---

### 2. Exact versus near-duplicate deduplication. What do you actually run, and what does it buy you?

`single-source`

<details>
<summary>Answer</summary>

Lee et al. (2021, *Deduplicating Training Data Makes Language Models Better*, arXiv:2107.06499) is the reference. They built two tools:

- **ExactSubstr** uses a suffix array to find repeated spans. If two documents share a substring of at least **50 tokens**, the span is removed from one of them. It catches boilerplate, license headers and copy-pasted paragraphs inside otherwise different pages.
- **NearDup** uses MinHash over n-gram sets to approximate Jaccard similarity, and flags whole documents that are near-copies: templated pages, lightly edited reposts, mirrored sites.

Their findings: in existing datasets, **over 1% of unprompted model output was copied verbatim from training data**; after dedup, models emitted memorized text **ten times less often** and reached the same or better accuracy in fewer training steps. One English sentence of 61 words appeared **over 60,000 times in C4**. Train-test overlap affected **over 4% of the validation set** of standard datasets, which ties dedup directly to question 7.

The nuance that shows experience comes from FineWeb (Penedo et al., 2024, arXiv:2406.17557). Deduplicating *globally* across all 96 Common Crawl snapshots hurt. For an old snapshot, the 10% of data that survived global dedup was **worse** than the 90% it removed: it had more ads, keyword lists and badly formatted text. Duplication across crawls turned out to be a signal of quality, because good pages persist and get re-crawled. Deduplicating each snapshot independently matched RefinedWeb's performance.

So the answer is not "dedup harder". Dedup at the granularity where duplicates are noise, and check the result with a training ablation rather than assuming fewer duplicates means better data. For SFT and preference sets the same tools apply at small scale: near-dup detection on prompts is the fastest way to find the one task a synthetic generator produced 400 times.

</details>

---

### 3. How does a model-based quality filter like FineWeb-Edu work, and what can go wrong with one?

`single-source`

<details>
<summary>Answer</summary>

The recipe from the FineWeb paper (arXiv:2406.17557):

1. **Label a sample with a strong LLM.** Llama-3-70B-Instruct scored 460,000 random pages for educational value on a 0 to 5 additive scale.
2. **Distil the labeler into a cheap classifier.** A linear regression head on a frozen Snowflake-arctic-embed-m encoder, trained on 410,000 of those labels and validated on the other 50,000.
3. **Choose a threshold by training ablations, not classifier metrics.** They kept documents scoring 3 or higher (F1 of 82% on the validation set at that threshold), because that gave the best trade-off between knowledge and reasoning benchmarks and others such as HellaSwag. Running the classifier over 15 trillion tokens cost 6,000 H100 GPU hours.

The output, FineWeb-Edu, is 1.3 trillion tokens and "dramatically" improves MMLU and ARC over other open web datasets, per the abstract.

The failure modes are the interview part:

- **You inherit the labeler's taste.** The classifier approximates Llama 3's view of "educational", with that model's biases about topic, dialect and register. Anything it undervalues quietly drops out of the corpus.
- **Filters trade breadth for target benchmarks.** The threshold was chosen because it helped MMLU and ARC without hurting HellaSwag too much. The chosen threshold was a trade-off between benchmarks, so a higher one costs something somewhere. Filtering towards the benchmarks you evaluate on also blurs the line with contamination (question 7).
- **Classifier accuracy is not the metric.** 82% F1 against LLM labels tells you it copies the labeler. Whether the filtered data trains a better model can only be found out by training small models on it.

The senior move is to treat the filter as a mixture decision (question 6): keep a slice of below-threshold data, or keep separate filters per domain, rather than one global cut.

</details>

---

### 4. What is model collapse, and does it mean you shouldn't train on synthetic data?

`single-source`

<details>
<summary>Answer</summary>

Shumailov et al. (*AI models collapse when trained on recursively generated data*, Nature, July 2024; preprint arXiv:2305.17493) showed that training each generation of model on the previous generation's output causes **irreversible defects in which the tails of the original distribution disappear**. Rare events, minority styles and unusual facts go first, and the model converges on a narrow, over-confident version of the data. They show it in VAEs, Gaussian mixture models and LLMs.

The condition matters. Gerstgrasser et al. (2024, *Is Model Collapse Inevitable?*, arXiv:2404.01413) confirmed collapse when synthetic data **replaces** real data each generation, and showed that when synthetic data **accumulates alongside** the original real data, collapse does not occur. In their linear-model analysis the test error then has a finite upper bound however many iterations run.

So the practical answer is "yes, but keep the real anchor":

- **Never let synthetic data replace your human data.** Mix it in and keep the original real set in every round.
- **Filter synthetic data with a verifier the generator doesn't control.** Unit tests, a checker, execution, or human spot checks. A generator grading its own output is the loop that collapses.
- **Measure diversity, not just quality.** Collapse shows up first as lost tails, so track the distribution of topics, lengths and rare categories across rounds, not only the average score.
- **Know that "real" web data is now part synthetic.** Any recent crawl already contains model-written text, so the accumulate-don't-replace rule applies to pretraining corpora you didn't generate yourself.

Where synthetic data is strongest is where correctness can be checked: code with tests, math with verifiable answers, tool calls that execute. Where it is weakest is where you use it to cover behaviour you haven't observed. It only contains what the prompt writer already imagined (question 8).

</details>

---

### 5. You're building preference pairs for DPO. How do you construct them, and what does label noise do?

`single-source`

<details>
<summary>Answer</summary>

DPO's loss (see [post-training.md question 3](post-training.md#3-sft-vs-rlhf-vs-dpoorpokto--whats-the-actual-difference-and-why-did-the-field-move)) pushes the policy towards `chosen` and away from `rejected` for every pair, with equal weight. Everything depends on the pairs.

**Construction choices that matter:**

- **Sample responses on-policy, or close to it.** Pairs built from a different model's outputs teach the policy to prefer text it would never produce. Sampling several responses from the current SFT model and ranking them gives pairs the policy can actually move between.
- **Make the pair differ in the thing you care about.** If chosen responses are systematically longer, the model learns length. Check chosen versus rejected length, format and refusal rate before training, and balance them.
- **Decide on the rater and log it.** UltraFeedback (Cui et al., 2023, arXiv:2310.01377) scaled to over 1 million GPT-4 ratings over 250k conversations, and needed deliberate steps to reduce annotation bias. AI feedback scales, but it brings the judge's biases with it (see [evaluation.md question 3](evaluation.md#3-name-the-ways-an-llm-judge-is-biased)).

**Label noise** is worse for DPO than for SFT, because a wrong pair actively trains the opposite preference instead of just adding a weak example. Wang et al. (2024, *Secrets of RLHF in Large Language Models Part II: Reward Modeling*, arXiv:2401.06080) name "incorrect and ambiguous preference pairs" as a primary obstacle. They measured preference strength by voting across several reward models, and found that pairs of different strength affect the reward model differently.

What to actually do:

- **Drop or down-weight ties and near-ties.** If raters or judges disagree, the pair carries more noise than signal.
- **Measure inter-rater agreement on a sample.** If two humans agree 65% of the time, no model will learn a sharper preference than that. Fix the rubric first.
- **Audit the highest-loss pairs after a first run.** Pairs the model persistently gets "wrong" are disproportionately mislabeled.

</details>

---

### 6. How do you choose a data mixture across domains?

`single-source`

<details>
<summary>Answer</summary>

The mixture (what fraction is web, code, math, books, your domain data) moves results as much as any single filter. The weak answer is "hand-tune by intuition". The stronger answer is that mixtures are chosen empirically with small proxy models, because intuition is often wrong.

Two published approaches worth knowing:

- **DoReMi** (Xie et al., 2023, arXiv:2305.10429) trains a 280M-parameter proxy with group distributionally robust optimization to produce domain weights, then trains an 8B model on the reweighted data. On The Pile it improved average few-shot downstream accuracy by **6.5 points** over the default weights and reached baseline accuracy in **2.6x fewer steps**, without knowing the downstream tasks.
- **RegMix** (Liu et al., 2024, arXiv:2407.01492) trains 512 tiny models (1M parameters, 1B tokens) on random mixtures, fits a regression from mixture to performance, and uses the best predicted mixture for a 1B model trained on 25B tokens. It matched or beat DoReMi with 10% of the compute. Its most quotable finding contradicts intuition: **web corpora, not "high-quality" sources like Wikipedia, correlated most strongly with downstream performance**, and domains interact in ways that often contradict common sense.

For fine-tuning rather than pretraining, the same logic applies at smaller scale:

- **Always include a replay slice** of general instruction data alongside the target domain, so the model doesn't forget what it could already do ([post-training.md question 4](post-training.md#4-what-is-catastrophic-forgetting-and-does-lora-actually-solve-it)).
- **Treat the ratio as a hyperparameter.** Sweep two or three ratios on a small run and evaluate on both the target task and a held-out general set.
- **Upsampling a small domain is repetition.** Repeating 10k examples 20 times behaves like training for 20 epochs on them, with the overfitting that brings.

</details>

---

### 7. How do you decontaminate a training set against your eval sets?

`foundational`

<details>
<summary>Answer</summary>

[model-evaluation.md question 7](model-evaluation.md#7-why-is-a-low-perplexity-on-a-public-benchmark-corpus-weak-evidence) covers *detecting* that an eval has leaked. Decontamination is the preventive side, and it belongs in the data pipeline, not in the eval report.

The standard method is n-gram overlap: tokenize every eval example, build the set of its long n-grams (13-grams is a common choice), and remove or flag any training document that shares one. The same suffix array and MinHash tools used for dedup (question 2) do this at scale. The 13-gram choice traces to the GPT-3 paper (Brown et al. 2020, arXiv 2005.14165, Appendix C), which decontaminated by filtering out training documents with a 13-gram overlap against its benchmarks. Lee et al. (arXiv:2107.06499) found train-test overlap in over 4% of the validation set of standard datasets. Leakage is common, not rare.

The judgment calls:

- **Decontaminate against every eval you will report, including internal ones**, and against their paraphrases where you can. Exact n-gram matching misses translated, reformatted or paraphrased copies, which is how benchmarks increasingly leak.
- **Do it at every stage, not just pretraining.** SFT and preference data built by prompting a model with "questions like these" can regenerate the benchmark. Synthetic data generated from eval-adjacent seeds is the easiest way to contaminate yourself.
- **Remove the document, not just the matched span.** A page that quotes a benchmark question usually discusses its answer nearby.
- **Keep a private, never-published held-out set** as the one eval you trust, because you cannot decontaminate against data you don't know has leaked into the base model.

The instinct: if a fine-tune jumps sharply on one benchmark and nowhere else, check contamination before celebrating.

</details>

---

### 8. How do you turn production traces into training data, from error analysis to a dataset, without creating privacy or licensing problems?

`single-source`

<details>
<summary>Answer</summary>

Production traces are the most valuable data you have, because they show the inputs users actually send and the failures you didn't anticipate. The path from trace to training data:

1. **Error analysis first.** Read a sample of traces, write down what went wrong in each, and group them into failure categories. [evaluation.md question 4](evaluation.md#4-how-much-of-a-genai-project-should-be-evaluation) covers how much time this deserves.
2. **Decide per category whether training is the fix.** Many failures are retrieval, tool or prompt bugs. Fine-tuning is for behaviour the model won't reliably produce otherwise ([post-training.md question 6](post-training.md#6-when-do-you-fine-tune-instead-of-using-rag--and-when-is-that-the-wrong-call)). Don't train around a broken tool.
3. **Build corrected examples for the categories that are.** The failing trace plus a corrected response makes an SFT example. The failing and corrected responses together make a preference pair (question 5), and that pair is on-policy because the rejected side is your model's real output.
4. **Hold part of each category out as eval** before any of it goes into training, and decontaminate the training set against it (question 7).
5. **Iterate.** Retrain, re-run error analysis, and check whether the category shrank without new ones appearing.

**Privacy and licensing hygiene is a gate, not a clean-up step:**

- **Check that you have the right to train on the data.** Terms of service, customer contracts and data-processing agreements often allow logging for debugging but not for training. It is a legal question, and it has to be answered before collection.
- **Redact, then verify the redaction.** Hamel Husain's evals FAQ (hamel.dev/blog/posts/evals-faq, fetched 2026-09-25) notes that redaction tools can miss sensitive information, so their output should be checked, and that edited traces must still preserve the behaviour you want to study. It ranks synthetic data as the last resort, after real and redacted traces. Weights memorize (question 2), so PII that reaches training can come back out in generations.
- **Track provenance per example.** Record the source, consent basis and license of every example, so a deletion request or license dispute can be traced to specific rows and the model retrained without them.
- **Watch the licences of scraped and generated data too.** Some model providers' terms restrict using outputs to train competing models — OpenAI's Terms of Use (openai.com/policies/terms-of-use, accessed 2026-09-25) prohibit using Output "to develop models that compete with OpenAI" — and code corpora carry per-file licences that automated detection gets wrong.

</details>
