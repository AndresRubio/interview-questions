# Evaluation

Reported as the single largest skill gap in the AI engineering candidate pool as of mid-2026, and interviewers are said to weight evaluation design **above** model-building — on the premise that failed LLM products almost always trace back to absent evaluation.

> **This file is *application* evaluation** — is your system answering correctly, is your judge validated. For *model* evaluation — perplexity, bits per byte, quantization damage — see [model-evaluation.md](model-evaluation.md). For *runtime* confidence on a single response — semantic entropy, calibration, selective answering — see [uncertainty-quantification.md](uncertainty-quantification.md). Knowing that these are three different layers is itself a thing interviews probe.

The depth expected here is higher than most candidates assume. "We used RAGAS" is not an answer.

---

### 1. What's the validation protocol for an LLM-as-judge?

`single-source`

<details>
<summary>Answer</summary>

Treat the judge as a classifier you have to **prove works**:

1. Label **100–200 examples per failure mode**
2. Split into **train / dev / test**, with roughly **30–50 Pass and 30–50 Fail in both dev and test**
3. Measure and report **True Positive Rate and True Negative Rate on the held-out test set**

Without this, a judge score is an unvalidated instrument and everything built on it is decoration.

This level of specificity is what the question is actually probing. Most candidates answer "you check it against human labels" and stop.

</details>

---

### 2. Binary labels or a 1–5 scale?

`single-source`

<details>
<summary>Answer</summary>

**Binary pass/fail.**

Adjacent points on a Likert scale are subjective and applied inconsistently across annotators, so the extra resolution is noise rather than information. Numeric labels are characterised as advanced and usually unnecessary.

If someone needs a graded output, derive it from the **proportion of binary passes** across a well-constructed set — not from asking a judge for a 3.5.

</details>

---

### 3. Name the ways an LLM judge is biased.

`single-source`

<details>
<summary>Answer</summary>

From a position paper applying measurement theory to LLM-as-judge (Chehbouni, Haddou, Cheung, Farnadi), which argues adoption has outrun evidence of validity:

- **Position bias** — the ordering of candidates in the prompt changes the verdict. Mitigate by randomising and swapping order.
- **Self-enhancement bias** — judges inflate scores for outputs from **models in their own family**. Never let a model grade its own lineage; always include a cross-family control.
- **Verbosity / length bias** — longer answers score higher independent of quality.
- **Adversarial brittleness** — superficial input changes can manipulate scores; universal attacks that inflate scores can be constructed, and pairwise preference evaluation is easy to game. One paper reports safety-judge misclassification driven **as high as 100%** under small modifications.

The deeper point, and the one that impresses: **human judgment isn't a clean gold standard either.** Human annotations in NLG evaluation are elicited inconsistently, so under high annotator uncertainty, reported LLM-judge/human correlations can be **artificially inflated**. Validating against humans is necessary but not sufficient.

</details>

---

### 4. How much of a GenAI project should be evaluation?

`single-source`

<details>
<summary>Answer</summary>

Far more than people expect. Hamel Husain and Shreya Shankar report **60–80% of total development time** going to error analysis and evaluation on real client LLM projects.

That number is useful in interviews because it reframes what a project *is* — most candidates present LLM work as mostly building.

**The highest-value single activity is manually reading traces.** Review at least ~100, and continue until new traces stop revealing new failure modes (theoretical saturation). The failure taxonomy that emerges is what your evals should be built from — not a vendor's default metric list.

On tooling: LangSmith, Arize and Braintrust are treated as roughly feature-equivalent commodities with no clear winner. The claimed highest-impact investment is building a **custom, domain-specific annotation tool** rather than adopting a vendor's built-in annotation UI.

</details>
