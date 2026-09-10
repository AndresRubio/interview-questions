# Evaluation playbook

The depth expected of a senior AI engineer when the conversation turns to evals. Mostly `single-source`, from practitioner write-ups and one measurement-theory position paper.

## Why this matters disproportionately

Evaluation is reported as **the single largest skill gap** in the AI engineering candidate pool as of mid-2026. Interviewers are said to weight evaluation-system design **above** model-building, on the premise that failed LLM products almost always trace back to absent evaluation.

Practitioners report **60–80% of total development time** going to error analysis and evaluation on real client projects.

## Split RAG evaluation in two

There is no single end-to-end score that substitutes for both layers.

- **Retrieval** → classical IR metrics: Recall@k, Precision@k, MRR. Cheap, no LLM in the loop, and where most real failures live.
- **Generation** → error analysis plus a **validated** LLM judge.

## Validating a judge

Treat it as a classifier you must prove works:

1. Label **100–200 examples per failure mode**
2. Split **train / dev / test** — roughly **30–50 Pass and 30–50 Fail in both dev and test**
3. Report **TPR and TNR on held-out test**

## Judge design rules

- **Binary pass/fail, not Likert.** Adjacent scale points are subjective and inconsistent across annotators. Numeric labels are advanced and usually unnecessary.
- **Cross-family control.** Never let a model grade its own lineage.
- **Randomise position** and swap order.

## Known judge biases

| Bias | Effect | Control |
|---|---|---|
| Position | Candidate ordering changes the verdict | Randomise / swap |
| Self-enhancement | Inflates scores for own model family | Cross-family judge |
| Verbosity | Longer answers score higher regardless of quality | Length-controlled comparisons |
| Adversarial brittleness | Superficial edits manipulate scores; safety-judge misclassification driven as high as **100%** | Adversarial test set |

The deeper point: **human judgment isn't a clean gold standard either.** Human annotations in NLG evaluation are elicited inconsistently, so under high annotator uncertainty, reported judge/human correlations can be **artificially inflated**.

## Error analysis

The highest-value single activity is **manually reading traces**.

- Review at least **~100 traces**
- Continue until **theoretical saturation** — new traces stop revealing new failure modes
- The failure taxonomy that emerges is what your evals get built from

## What to skip

**Don't ship generic framework metrics.** Off-the-shelf metrics shipped by default in eval frameworks are argued to be *actively harmful* when used as quality measures — they waste time and create false confidence. Similarity metrics such as ROUGE and BERTScore are not useful for evaluating LLM outputs in most applications (stated exception: search and recommendation).

A `faithfulness: 0.87` in a README invites the question you don't want: *what does that number mean, and how do you know?*

## Tooling

LangSmith, Arize and Braintrust are treated as roughly **feature-equivalent commodities** with no clear winner. The claimed highest-impact investment is building a **custom, domain-specific annotation tool** rather than adopting a vendor's annotation UI.

## Agent-specific eval hazards

- **Lucky passes** — up to **23.2%** of SWE-agent passes across 2,614 OpenHands trajectories, enough to shift rankings by five positions
- **Environment sensitivity** — **6+ percentage-point** swings from container resource configuration alone
- **Contamination** — a model documented identifying a benchmark by name and decrypting its answer key

Controls: network isolation, pinned and reported container resources, trajectory inspection over pass/fail, variance across seeds.

## Building a gold set cheaply

Where a corpus has authoritative structure, generate **known-item** retrieval questions by construction: for each unit, a question whose answer is that exact unit. Ground truth known by definition — Recall@k and MRR with no labelling.

**State the limitation before anyone asks:** questions generated *from* a passage share vocabulary with the target and are systematically easier than real questions. Such a set measures **regressions** well and **absolute quality** badly. Mitigate with a small hand-written holdout of genuinely hard questions.
