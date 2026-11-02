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

---

### 5. Build an eval suite for a customer support bot from nothing.

`single-source`

<details>
<summary>Answer</summary>

I would start by reading transcripts, and I would hold off on choosing metrics until I had. Pull a few hundred real conversations (from the bot's pilot, or from human agents' chat logs if the bot isn't live yet), read at least 100 of them, and write a note on each one that went wrong. Hamel Husain's evals FAQ (hamel.dev/blog/posts/evals-faq, fetched 2026-11-02) calls error analysis the most important activity in evals and recommends reviewing at least 100 traces, continuing while new failure types still appear (question 4). Group the notes into a taxonomy. For support bots it usually looks something like this: wrong policy facts, a refund or credit granted outside the rules, a wrong or missing tool call (looked up the wrong order, never checked the account), no handoff when the user asked for a human or was clearly stuck, an off-topic or unsafe answer, and losing context across turns. A metric list picked before this step measures what a vendor thought mattered, and the FAQ argues that generic metrics used as quality measures give false confidence.

**Turning the taxonomy into cases.** Each failure mode becomes a slice of the dataset:

- Golden questions: the common intents (order status, returns, password reset, billing) with a reference outcome. Take them from real traffic so the wording is real.
- Adversarial and out-of-scope cases: prompt injection in the user message, requests for another customer's data, legal or medical questions, competitor comparisons. The pass condition is usually a refusal or a redirect.
- Escalation and handoff: angry users, repeated failed attempts, explicit "let me talk to a person", account-security issues. The bot passes if it hands off at the right moment and passes a usable summary to the human.
- Policy and refund rules: boundary cases on purpose. Day 29 and day 31 of a 30-day return window, an item that is excluded from returns, a refund amount just over the agent's limit.
- Multi-turn conversations: the user changes the order number halfway through, or corrects themselves, or the needed fact came three turns earlier. Many failures only show up here, and single-turn cases will miss them.

For multi-turn cases you need a simulated user, because a fixed script breaks as soon as the bot answers differently. τ-bench (Yao, Shinn, Razavi, Narasimhan, Sierra, arXiv:2406.12045) does this for retail and airline support: an LLM plays the user from a scenario, the agent works against tools and a database, and success is judged mainly by comparing the final database state to the expected one (plus required facts in the agent's replies). That last idea transfers directly. Check what the bot did (refund issued, address changed) as well as what it said.

**Picking the check for each case.** Use the cheapest check that is reliable:

- Deterministic checks wherever there is a right answer: the tool called and its arguments, the final state of the order or ticket, the refund amount, whether a handoff fired, whether the response quotes the correct return window. These are fast and leave no room for judge error.
- An LLM judge for the fuzzy parts: tone, whether the answer actually solved the problem, whether the handoff summary is usable. One binary judge per failure mode, validated against human labels with TPR and TNR on a held-out set (questions 1 and 2) before its scores are trusted.
- Human review for what neither can cover: a weekly sample, new intents, and every case where the judge and a deterministic check disagree. This also keeps the judge calibrated.

**Slice by intent, and measure reliability.** Report pass rates per intent and per failure mode instead of one overall number. A 92% overall can hide a refund intent at 60%, and refunds are where mistakes cost money. Run each case several times, too. τ-bench's pass^k is the probability that all k independent attempts at a task succeed, and the paper reports GPT-4o at under 25% pass^8 on retail even though its single-attempt success was higher. A support bot talks to thousands of customers, so a task it gets right four times out of five is a task it gets wrong for a fifth of them.

**Running it as a regression gate.** The suite runs in CI on every change to the prompt, model, tools or retrieval index. Split it into a fast tier (deterministic checks plus a small judged set, a few minutes) on every pull request, and a full tier with repeated trials and the multi-turn simulator before release. Fail the build on a drop in any high-stakes slice (policy, refunds, escalation, security), and not only on the average. On size: start with roughly 100 to 200 cases built from the taxonomy, enough for each major intent to have at least 20 to 30 cases so a per-slice rate means something. A pass rate on 20 cases has a wide confidence interval, so treat a one-case change as noise and gate on larger drops or on any failure in a must-pass case. Add the 100 to 200 labelled examples per judged failure mode that judge validation needs. How production failures feed into this set without skewing it, and how the set is versioned, is covered in [llmops.md question 7](llmops.md#7-how-do-production-traces-become-eval-cases-without-polluting-the-eval-set).

**Connecting offline to online.** Offline evals predict, and production confirms. The online signals for support are CSAT, escalation rate, containment (the share of conversations resolved without a human), reopen rate (did the customer come back about the same issue within a few days) and repeat contact. The general online layering is in [llmops.md question 2](llmops.md#2-offline-evals-pass-why-do-you-still-need-online-evaluation-and-what-does-it-measure). What is specific here is that containment is easy to game. A bot that never offers a handoff, that buries the "talk to a human" option, or that lets a frustrated user give up and close the chat counts all of those as contained. Containment goes up while customers are worse off. So I'd never report containment without the guards that catch this: reopen and repeat-contact rate, CSAT on contained conversations specifically, and a sampled check of "contained" conversations for users who abandoned or asked for a human and didn't get one. If containment rises and the offline escalation slice starts failing, that's the bot refusing to hand off.

A senior answer ends with ownership. Someone owns the taxonomy and re-reads fresh transcripts every few weeks, because new products and policy changes create failure modes the suite has never seen.

</details>
