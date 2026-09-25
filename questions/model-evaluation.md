# Model evaluation

Intrinsic metrics — perplexity, cross-entropy, bits per byte — and the benchmark hygiene around them.

**This is a different layer from [evaluation.md](evaluation.md).** That file covers *application* evaluation: does your RAG system answer correctly, is your judge validated. This one covers *model* evaluation: how good is the language model at modelling language. Conflating the two is itself a common interview mistake.

All `foundational` — established material, not output of the verification pipeline used elsewhere in this repo.

---

### 1. Define perplexity precisely.

`foundational`

<details>
<summary>Answer</summary>

Perplexity is the exponentiated average negative log-likelihood per token:

```
PPL(X) = exp( -(1/N) * Σ log p(x_i | x_<i) )
```

Equivalently, it is **the exponential of cross-entropy**:

```
PPL = exp(H)        where H is cross-entropy in nats
PPL = 2^H           where H is cross-entropy in bits
```

**Interpretation:** the effective number of equally-likely options the model is choosing among at each step — a weighted average branching factor. A perplexity of 10 means the model is, on average, about as uncertain as if it were picking uniformly from 10 candidates.

Lower is better. The floor is 1 (perfect prediction); a uniform distribution over a vocabulary of size V gives PPL = V.

The thing to say that shows you understand it rather than memorised it: **perplexity is not a separate metric from the training loss.** It's a monotonic transform of the cross-entropy that language models are trained to minimise. That's precisely why it's excellent for monitoring training and poor for judging products.

</details>

---

### 2. Why can't you compare perplexity between two models with different tokenizers?

`foundational`

<details>
<summary>Answer</summary>

Because perplexity is **per token**, and the token is not a fixed unit.

A model with a larger or more efficient vocabulary encodes the same passage in fewer tokens. Each of those tokens therefore carries more information, and predicting it is harder — so its per-token perplexity is *higher* even if it models the text strictly better. Run the argument to its limit: a character-level model has very low per-token perplexity and that number is meaningless next to a BPE model's.

**The fix is to normalise by a tokenizer-independent unit.** Divide total negative log-likelihood in bits by the number of UTF-8 bytes (or characters) in the original text:

```
BPB = ( -Σ log2 p(x_i | x_<i) ) / (number of bytes in the raw text)
```

**Bits per byte** (or bits per character) *is* comparable across tokenizers, which is why serious pretraining papers report it.

Three other things must match before any perplexity comparison is meaningful:

- **The evaluation corpus.** Perplexity is a property of model *and* text.
- **Context length**, and the **stride** if you're using a sliding window over long documents. A stride equal to the window is cheap and pessimistic; a stride of 1 is expensive and optimistic. Papers that don't state their stride aren't comparable.
- **Whether the text was in training data.** See question 7.

</details>

---

### 3. Perplexity went down but the product got worse. Explain how that happens.

`foundational`

<details>
<summary>Answer</summary>

Several distinct mechanisms, and naming more than one is the point:

- **Perplexity measures next-token likelihood on a corpus distribution.** Your product cares about instruction-following, output format, refusal behaviour, tool-call validity and factuality in free generation. None of those is directly measured.

- **The alignment tax.** Instruction tuning and RLHF typically *increase* perplexity on raw text corpora while making a model dramatically more useful. A model with worse perplexity can be the better product — so a perplexity regression is not automatically a quality regression.

- **Perplexity is teacher-forced.** It's computed with ground-truth prefixes: the model is scored on predicting token *i* given the *correct* tokens 1..i−1. Real generation conditions on its own previous outputs, where errors compound. Perplexity never observes that failure mode at all.

- **Distribution mismatch.** Perplexity on WikiText tells you very little about Spanish legal text, or your support tickets.

- **Averaging hides the tail.** It's a mean over tokens. A model can be marginally better on the vast bulk of easy tokens and much worse on the rare ones that carry the meaning.

The synthesis worth offering: **perplexity is a good regression detector and a bad quality measure.** Use it to notice that something broke; never use it to argue something is good.

</details>

---

### 4. So when *is* perplexity the right tool?

`foundational`

<details>
<summary>Answer</summary>

It has real, specific uses — dismissing it entirely is as wrong as over-trusting it:

- **Pretraining monitoring and scaling laws.** It *is* the objective. Cheap, smooth, and the basis of compute-optimal scaling work.
- **Detecting catastrophic degradation.** A botched quantization, a bad model merge, a broken fine-tune — these show up immediately as a perplexity blow-up.
- **Domain adaptation sanity checks.** Did continued pretraining on your corpus actually lower perplexity on held-out in-domain text?
- **Comparing checkpoints within one model family** — same tokenizer, same eval set, same context handling. This is the only truly clean comparison.

Its two structural advantages over benchmark accuracy are worth stating explicitly:

1. **No labels required.** Any held-out text works.
2. **Very low variance.** Benchmark accuracy over a few hundred items is noisy; perplexity over millions of tokens is stable, so small genuine changes are detectable.

That combination is exactly what you want in a **CI regression gate** — which is the bridge to how you'd actually use it in a production system.

</details>

---

### 5. You're quantizing a model for cheaper serving. How do you measure the damage?

`foundational`

<details>
<summary>Answer</summary>

Perplexity delta on a held-out set is the standard smoke test, and it is **necessary but not sufficient**. A quantization can show a negligible perplexity change while behaving materially worse, because the average is dominated by easy tokens.

A stronger measure: **KL divergence between the quantized model's output distribution and the full-precision model's, on identical inputs.**

```
D_KL( p_fp16(· | context)  ||  p_quant(· | context) )
```

This is more sensitive than perplexity because it compares the model against *the model you're trying to preserve* rather than against text. Perplexity can stay flat while the distribution shifts substantially — KL catches that.

Then layer on what perplexity structurally cannot see:

- **Downstream task evals** on the tasks you actually run
- **Long-context behaviour** — degradation is often non-uniform across position
- **Structured output and tool-call validity** — a small drop in precision on low-probability tokens breaks JSON and schema conformance disproportionately
- **Reasoning chains**, where a single derailed token compounds in a way a per-token average never reflects

The general principle: quantization damage is **non-uniform**, and every aggregate metric hides non-uniformity.

</details>

---

### 6. Distinguish intrinsic from extrinsic evaluation.

`foundational`

<details>
<summary>Answer</summary>

- **Intrinsic** — measures the model against its own objective, with no task and no labels. Perplexity, cross-entropy, bits per byte.
- **Extrinsic** — measures performance on something you care about. Benchmark accuracy, human preference, task success, application metrics.

The failure mode is treating intrinsic as a proxy for extrinsic. They **correlate during pretraining within a single model family**, and they **decouple** in exactly the situations where you most want an answer:

- After alignment (the alignment tax)
- Across model families with different tokenizers and data mixes
- At the top of the range, where the remaining perplexity differences are tiny and capability differences are not

A useful way to put it in an interview: perplexity answers *"how surprised is this model by text like this?"* Nobody ships a product whose success criterion is the model's surprise.

</details>

---

### 7. Why is a low perplexity on a public benchmark corpus weak evidence?

`foundational`

<details>
<summary>Answer</summary>

**Contamination.** If the evaluation text was in the pretraining data, low perplexity measures memorisation, not modelling — and public corpora are the most likely things to have been scraped.

This is worse for perplexity than for multiple-choice benchmarks, because perplexity scores the *exact token sequence*. Near-verbatim memorisation drives it down sharply, and unlike a benchmark answer there's no shuffling or paraphrasing to disrupt it.

Mitigations, roughly in order of strength:

- **Evaluate on text that postdates the training cutoff.** The only robust defence.
- **Hold out genuinely private, in-domain data** that was never published.
- **n-gram overlap checks** between the eval set and training data, where you have access to the training corpus.
- **Canary strings** — deliberately planted unique sequences that reveal whether a corpus was ingested.

The connected point that generalises beyond perplexity: any evaluation whose data could plausibly be in the training set is measuring recall of the answer rather than the capability. See [agent-architecture.md](agent-architecture.md) for the documented case of a model identifying a benchmark by name and decrypting its answer key.

</details>
