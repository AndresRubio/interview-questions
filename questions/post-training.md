# Fine-tuning and post-training

LoRA, QLoRA, preference optimization, and the build-vs-buy question of fine-tuning against a base model versus retrieving context at inference time. This sits next to [retrieval-and-rag.md](retrieval-and-rag.md) — question 6 here is the other half of that decision — and next to [model-evaluation.md](model-evaluation.md), since a botched fine-tune is one of the perplexity blow-ups that file's question 4 is watching for. It's a different layer from [uncertainty-quantification.md](uncertainty-quantification.md): that file asks whether a single response can be trusted at inference time; this one asks how the weights got the way they are.

Sourcing note: the question list below matches what current (2025-2026) prep guides converge on independently — none of them ties a question to a named company, so treat these as the commonly asked set, not confirmed transcripts from a specific employer.

---

### 1. Explain LoRA. What do rank and alpha actually control?

`verified 3-0`

<details>
<summary>Answer</summary>

LoRA freezes the pretrained weight matrix `W` and learns an additive low-rank update instead of training `W` directly:

```
W' = W + ΔW = W + BA
```

where `B` is `d × r`, `A` is `r × k`, and the rank `r ≪ d, k`. Only `A` and `B` are trained; `W` never moves. Because `r` is small, this trains **under 1% of the parameters** of the layer it's attached to — the original paper reports up to 10,000x fewer trainable parameters and 3x less GPU memory versus full fine-tuning GPT-3 175B, with quality on par with or better than full fine-tuning on the benchmarks it tested (Hu et al., 2021, *LoRA: Low-Rank Adaptation of Large Language Models*, arXiv:2106.09685).

**Rank (`r`)** sets the capacity of the update — how much new information the adapter can encode. Common starting points are 8, 16, or 32. Push it too low and the adapter can't represent what the task needs; push it too high and you're approaching full fine-tuning's parameter count and its overfitting risk, while giving up LoRA's efficiency advantage for no clear gain.

**Alpha (`α`)** is a scaling factor applied to the update, typically as `ΔW · (α/r)`. It controls how strongly the adapter's output is weighted relative to the frozen base — practically, it's a learning-rate-like knob. The common heuristic is to scale `α` with `r` (e.g., keep `α/r` fixed, or set `α = 2r`) rather than tuning them independently.

The deployment payoff worth naming: because `ΔW` merges additively into `W`, you can fold the adapter into the base weights after training and get **zero additional inference latency** — unlike adapter-layer approaches that add depth to the forward pass.

</details>

---

### 2. QLoRA vs. plain LoRA — when do you reach for each?

`verified 3-0`

<details>
<summary>Answer</summary>

QLoRA backpropagates through a **frozen base model quantized to 4-bit** (NF4, a data type built for normally-distributed weights) into LoRA adapters trained in higher precision, plus double quantization of the quantization constants and paged optimizers to smooth memory spikes (Dettmers et al., 2023, *QLoRA: Efficient Finetuning of Quantized LLMs*, arXiv:2305.14314). The headline result: a 65B model fine-tuned on a single 48GB GPU, with the resulting model reaching 99.3% of ChatGPT's performance on the Vicuna benchmark in the paper's own evaluation.

The decision rule interviewers want:

- **Reach for QLoRA when the base model doesn't fit your GPU in full or half precision at all.** It's the difference between fine-tuning a 70B model on one GPU versus needing a multi-GPU cluster.
- **Reach for plain LoRA when memory isn't the constraint.** QLoRA trains more slowly, because every forward pass pays a dequantization cost, and it can lose a small amount of quality relative to full-precision LoRA — the 4-bit base is a lossy approximation of the original weights, even though the adapters themselves stay high-precision.

The senior instinct: QLoRA is not "free" efficiency — it's a memory-for-speed-and-a-little-quality trade, and you only make that trade when memory is actually the binding constraint, not by default.

</details>

---

### 3. SFT vs. RLHF vs. DPO/ORPO/KTO — what's the actual difference, and why did the field move?

`verified 3-0`

<details>
<summary>Answer</summary>

**SFT (supervised fine-tuning)** trains directly on demonstrations — prompt/response pairs a human (or a stronger model) wrote. It teaches the model *what a good answer looks like* but has no notion of *how much better one answer is than another*.

**RLHF**, as InstructGPT (Ouyang et al., 2022, arXiv:2203.02155) established the recipe: SFT first, then train a **separate reward model** on human pairwise preference rankings, then optimize the policy against that reward model with PPO, usually with a KL penalty back to the SFT model to keep it from drifting too far. It works — InstructGPT's 1.3B model was preferred over 175B GPT-3 — but it's a three-stage pipeline with a reward model that can be gamed (reward hacking) and PPO's well-known instability and hyperparameter sensitivity.

**DPO** (Rafailov et al., 2023, *Direct Preference Optimization: Your Language Model is Secretly a Reward Model*, arXiv:2305.18290) collapses reward modeling and RL into one supervised-style loss: it shows the optimal RLHF policy has a closed form in terms of the reward, which lets you substitute preference pairs directly into a classification-style objective on the policy itself — no reward model, no RL loop, no PPO instability. **ORPO** and **KTO** are further simplifications in the same family: ORPO folds preference alignment into the SFT step itself (no separate reference model needed), and KTO drops the requirement for paired preferences entirely, learning from unpaired binary "good/bad" signal, which matters because that kind of data is far cheaper to collect than ranked pairs.

The trade-off to state out loud: DPO-family methods are simpler to implement and cheaper to run, but they optimize directly against a fixed offline preference dataset — RLHF's reward model can in principle generalize and keep providing signal as the policy explores on-policy. In practice, most 2025-2026 stacks default to DPO or a variant unless they have specific reasons to want a reward model (e.g., online RL against live signal, which is where RLHF-style setups persist for agentic and reasoning training — see question 7).

</details>

---

### 4. What is catastrophic forgetting, and does LoRA actually solve it?

`verified 3-0`

<details>
<summary>Answer</summary>

Catastrophic forgetting is what happens when fine-tuning on a new task or domain degrades performance on capabilities the base model already had — the gradient updates that make it better at the target task overwrite representations the general model relied on elsewhere.

The common interview answer stops at "LoRA fixes this because it doesn't touch the base weights." That's incomplete, and a controlled study catches it: **Biderman et al., 2024, *LoRA Learns Less and Forgets Less*, arXiv:2405.09673**, compared LoRA against full fine-tuning on code and math, across both instruction fine-tuning and continued pretraining. Their finding cuts both ways — say both halves, not just the flattering one:

- **LoRA substantially underperforms full fine-tuning on the target task**, particularly on code and math, in standard low-rank settings.
- **LoRA forgets less on out-of-domain tasks** than full fine-tuning, and forgets less than common regularizers like weight decay or dropout — it also preserves more diverse generations.

So LoRA doesn't eliminate the learn/forget trade-off, it *shifts* it: you're trading target-task ceiling for base-capability retention. That's a real and useful property — it's exactly why LoRA is the default for narrow adapters you expect to stack or swap — but it's not "no forgetting," and it's not free performance on the task you actually fine-tuned for.

Mitigations beyond "use LoRA" worth naming: replay a slice of the original pretraining/instruction distribution during fine-tuning, keep the adapter's rank and learning rate conservative rather than pushing for target-task ceiling, and evaluate on out-of-domain held-out sets specifically to catch regression — not just the fine-tuning task's own metric.

</details>

---

### 5. You've trained a LoRA adapter. How do you deploy and version it in production?

`foundational`

<details>
<summary>Answer</summary>

This is a systems question disguised as a fine-tuning question, and it's where a lot of candidates who can explain the math have nothing to say.

**Merge vs. keep-separate** is the first fork. Merging `ΔW` into `W` gives you a single dense checkpoint with zero extra inference latency or serving complexity — but you lose the ability to swap adapters per-request, and you pay full storage/deployment cost per variant. Keeping the adapter separate (a few MB instead of a full checkpoint) lets a single served base model swap in different adapters per request — this is what makes multi-tenant, per-customer, or per-task fine-tunes economical, and it's the design several LoRA-serving systems (e.g., S-LoRA-style batched adapter serving) are built around.

**Versioning** needs the same discipline as any model artifact: pin the adapter to the exact base model checkpoint and tokenizer it was trained against (an adapter trained against one base checkpoint is not guaranteed compatible with a later one, even a minor version bump), record the training data snapshot and hyperparameters (rank, alpha, learning rate, epochs) alongside the weights, and gate promotion behind the same eval suite the base model uses — plus a regression check against the out-of-domain capabilities question 4 says you're risking.

**Rollback and blast radius** matter more with adapters than people expect precisely because they're cheap to ship: it's easy to fall into shipping adapter updates without the release discipline a full model deploy would get. Treat an adapter swap as a deploy: canary it, monitor the same production metrics you'd watch for a base model change, and keep the previous adapter version one config flip away.

</details>

---

### 6. When do you fine-tune instead of using RAG — and when is that the wrong call?

`foundational`

<details>
<summary>Answer</summary>

Fine-tuning and RAG solve different problems, and the interview mistake is treating them as competing solutions to "the model doesn't know X."

**RAG is for knowledge** — facts, documents, anything that changes over time or is too large to bake into weights. It's cheap to update (edit the index, not retrain), it's auditable (you can point to the retrieved source), and it composes with any base model. Its failure modes are retrieval failures: irrelevant or missing context, and the model still has to *use* what it's given correctly — see [retrieval-and-rag.md](retrieval-and-rag.md) for where that breaks in practice.

**Fine-tuning is for behavior** — teaching the model a response format, a tone, a domain-specific reasoning pattern, a tool-use convention, or compressing a long, expensive prompt (few-shot examples, a long system prompt) into the weights so you don't pay for it in every context window. It's the right tool when the thing you're trying to change is *how the model behaves*, not *what it knows*.

The question that actually separates strong from weak candidates: **can you name a case where the two combine?** A common production pattern is fine-tuning a model to reliably *use* retrieved context — follow citation format, refuse when retrieval comes back empty, weight recency correctly — while RAG still supplies the facts. Treating them as mutually exclusive, or defaulting to fine-tuning because it "feels more custom," is the wrong instinct: fine-tuning is slower to iterate on, doesn't fix stale or missing knowledge, and (per question 4) risks degrading capabilities you didn't mean to touch. Reach for it when RAG genuinely can't express the change you need — a behavior, not a fact.

</details>

---

### 7. What's the current failure mode in distilling and RL-training agentic models, beyond classic SFT distillation?

`single-source`

<details>
<summary>Answer</summary>

The classic distillation story — train a small "student" on a large "teacher"'s outputs to compress capability — gets more fragile once the target isn't a single response but an **agentic trajectory** inside a harness (the scaffolding: tool definitions, retry logic, planning prompts) that's been optimized for a specific model.

A 2026 Salesforce paper makes this concrete: when a harness has been evolved/optimized around a strong model, naively training a weaker model on that strong model's complete expert trajectories **backfires** — performance regressed 4 to 30 points across all seven tasks tested. The mechanism: the weaker model imitates the expert's *planning strategy* without the competence to execute it, and that strategy no longer matches the harness that was tuned for the weaker model's own native planning style — imitation and harness fit come apart (*Co-Evolving Harnesses and Models: On-Policy Correction Helps Weaker Models Catch Up Where Imitation Fails*; arXiv 2609.09134, preprint). Their fix is on-policy: keep the weaker model's own trajectories as the backbone, and only inject expert correction selectively at points where it actually fails, rather than replacing its whole planning style.

The generalizable point for an interview answer: **distillation quality depends on match between student capability, teacher behavior, and the harness the trajectories run inside** — copying outputs (SFT-style distillation) is not the same as transferring competence, and this is sharper for multi-step agentic behavior than for single-turn text, because a harness mismatch compounds across steps the way single-token errors compound in autoregressive generation (see [model-evaluation.md](model-evaluation.md) question 3 on teacher-forcing for the parallel). If you're asked how you'd validate a distilled or RL-trained agentic model before shipping it, the answer this paper argues for is: evaluate on-policy, inside the harness it will actually run in, not against the teacher's trajectories in isolation.

</details>
