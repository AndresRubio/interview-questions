# Inference and systems

Serving infrastructure is live system-design material for AI engineering roles, not background reading. At least one frontier lab runs a system design round specifically on LLM inference under variable load.

---

### 1. "Design a Claude chat service." Where does this interview actually go?

`single-source`

<details>
<summary>Answer</summary>

Straight into serving infrastructure: **request batching, queuing, and GPU utilisation under variable load.**

Cover:

- **Continuous vs static batching**, and why continuous wins on throughput
- **KV cache as the real memory constraint**, and how it bounds achievable concurrency
- **Admission control and queueing** when demand exceeds capacity
- **Separating prefill from decode** — they have different bottlenecks and different scaling behaviour
- **Tail latency consequences** of batching harder

Have the metrics ready as first-class design requirements, not afterthoughts: **TTFT, TBT, tokens/second, cost per user.**

</details>

---

### 2. Do the cost arithmetic out loud: 50,000 daily users, an agent averaging 12 LLM calls per session, 4K input and 500 output tokens per call.

`single-source`

<details>
<summary>Answer</summary>

Senior AI engineer interviews expect **explicit arithmetic**, unprompted.

Per session: `12 x 4K = 48K` input tokens, `12 x 500 = 6K` output tokens.

Daily across 50K users: **2.4B input**, **300M output**.

At an illustrative $3/M input and $15/M output:

```
input:   2,400 x $3  = $ 7,200
output:    300 x $15 = $ 4,500
                       -------
total                = $11,700 / day
```

Roughly **$4.3M annually**, or **~$0.23 per user per day**.

Then attack the biggest term, which is the actual point:

- **Prompt caching** on the shared prefix — usually the dominant win, since input dwarfs output in agent workloads and system prompts plus tool definitions repeat every turn
- **Reduce calls per session** — 12 is a design choice, not a constant
- **Route easy turns to a smaller model**
- **Cap context growth**

</details>

---

### 3. Which inference optimisations would you reach for, and what does each cost you?

<details>
<summary>Answer</summary>

Every one is a trade, and naming the cost is the answer:

| Technique | Win | Cost |
|---|---|---|
| Quantization | Memory and throughput | Some accuracy; loss is task-dependent, so measure rather than assume |
| Continuous batching | Large throughput gain | Tail latency for individual requests |
| KV cache management / paged attention | Higher achievable concurrency | Implementation complexity |
| Speculative decoding | Lower latency when the draft agrees often | Wasted compute when it doesn't |
| Constrained decoding / structured output | Guaranteed parseable output | Can distort the distribution and hurt quality if the schema fights the model |
| Prompt caching | Usually the cheapest large win in agent workloads | Requires stable prefixes — cache-busting on every turn defeats it |

</details>

---

### 4. How do you make a GenAI system observable in production?

`single-source`

<details>
<summary>Answer</summary>

Trace every LLM call and tool call as spans, using the **OpenTelemetry GenAI semantic conventions** so the data isn't vendor-locked. Those conventions are still in Development status and can still change (see [llmops.md question 1](llmops.md#1-how-would-you-trace-an-agent-run-end-to-end-and-what-is-the-actual-status-of-the-opentelemetry-genai-conventions)), so "not vendor-locked" is the goal, not yet a guarantee — pin the semconv version and wrap instrumentation behind a thin layer of your own.

Capture: prompt and completion token counts, model and version, latency split into **TTFT and total**, tool name and outcome, and a stable trace id linking back to the user-visible interaction.

Then the part people forget: a **regression suite that runs on every prompt or model change**, plus **sampled trace review as a standing practice** rather than incident response.

The framing that lands: **prompt changes are code changes and need the same gate.** A team that reviews code but ships prompt edits straight to production has an obvious hole, and saying so demonstrates you've operated one of these systems rather than just built one.

</details>

---

### 5. You're asked to self-host Llama-3-70B for 32 concurrent users at 8K context. How many GPUs?

`single-source`

<details>
<summary>Answer</summary>

Size **two** things, not one. The mistake is sizing for the weights and then running out of memory on the KV cache.

**Weights** = parameters × bytes per parameter. That's 2 bytes at FP16/BF16, 1 at FP8/INT8, and about 0.5 at 4-bit (AWQ/GPTQ). So 70B is ~140 GB at FP16, ~70 GB at FP8 and ~35 GB at INT4.

**KV cache per token** = `2 (K and V) × layers × KV heads × head dim × bytes`. Llama-3.1-70B's config has 80 layers and 8 KV heads (grouped-query attention, against 64 query heads), with head dim 8192 / 64 = 128:

```
2 × 80 × 8 × 128 × 2 bytes = 327,680 bytes ≈ 0.33 MB per token
32 users × 8,192 tokens   = 262,144 tokens
262,144 × 0.33 MB        ≈ 86 GB of KV cache
```

That's roughly the size of the FP8 weights again. FP8 weights plus a full cache come to ~156 GB, before activation and runtime overhead. That's more than one H200 (141 GB) and more than two 80 GB H100s once you add headroom. Your options:
- Quantize the KV cache too.
- Accept fewer concurrent full-length contexts. Paged attention only allocates what a request actually uses (PagedAttention, Kwon et al., arXiv 2309.06180).
- Add a GPU.

The senior move is saying the cache number out loud before anyone asks. GQA is the reason it's 86 GB and not 690 GB, because full multi-head attention would cache all 64 heads.

**MoE caveat.** Mixture-of-experts models compute with only their active parameters but must *hold* all of them. They're cheap per token but still need a node big enough for the full weights.

</details>

---

### 6. Why is single-user decode so slow on a GPU that costs this much, and what fixes it?

`single-source`

<details>
<summary>Answer</summary>

The two phases have different bottlenecks:

- **Prefill** (processing the prompt) handles all prompt tokens in parallel. It is **compute-bound**.
- **Decode** (one token at a time) has to read every weight from memory for each token. It is **memory-bandwidth-bound**.

That gives a back-of-envelope ceiling for a single stream: `tokens/s ≤ memory bandwidth / bytes of weights`. An H100 SXM has 3.35 TB/s (NVIDIA spec), so a 70 GB FP8 model tops out around **48 tokens/s for one user**, however many FLOPs sit idle. This is an upper bound; real numbers come in lower.

The fix is **batching**. One read of the weights serves every sequence in the batch, so aggregate throughput rises almost linearly with batch size. That holds until you either become compute-bound or run out of KV cache memory, which ties this back to [question 5](#5-youre-asked-to-self-host-llama-3-70b-for-32-concurrent-users-at-8k-context-how-many-gpus). Continuous batching and paged KV caches in vLLM, SGLang and TensorRT-LLM exist to keep that batch full.

**Speculative decoding** attacks the same bottleneck from a different side. A small draft model proposes several tokens, and the large model verifies them in one pass. The original paper reports 2–3× speedups on T5-XXL with **identical outputs** (Leviathan, Kalman & Matias, arXiv 2211.17192). The gain depends on how often the draft model is accepted, so measure it on your traffic.

The trade-off interviewers want named: **a strict latency SLA forces smaller batches, which raises cost per token.** Throughput and per-user latency pull against each other, and the SLA picks the point.

</details>

---

### 7. Turn a GPU bill into a cost per million tokens. What dominates it?

`single-source`

<details>
<summary>Answer</summary>

```
$/1M tokens = GPU $/hour ÷ (aggregate tokens/s × 3,600) × 1,000,000
```

Here's a worked example with illustrative prices (rental prices move fast). Two H100s at ~$3/hour each is $6/hour. Assume they serve a 70B FP8 model at ~2,000 tokens/s aggregate. That's 7.2M tokens an hour, or **~$0.83 per million tokens at full utilization**.

That last qualifier is the whole answer. **Utilization is the biggest lever.** You pay for the GPU whether requests arrive or not, so at 10% utilization the same setup costs ~10× more per token. Then add what the formula hides:
- **Redundancy.** Production usually means at least two replicas, which doubles the minimum spend.
- **Idle capacity overnight**, and the headroom you keep for peaks.
- **Engineering time, monitoring, autoscaling, and model loading.** On serverless GPUs, a cold start has to load tens of GB of weights.

The pricing model shifts who carries the utilization risk:
- **On-demand** is flexible and most expensive.
- **Reserved capacity** is cheaper, but idle hours are yours to pay for.
- **Spot** is cheapest but can be interrupted. It's fine for batch work and risky for live serving.
- **Serverless GPU** scales to zero, but you pay in cold starts.
- **Managed open-model APIs** (Together, Fireworks, Bedrock) sell per token and absorb the risk for you.

For the token-side cost levers on APIs, see [cost-and-caching.md](cost-and-caching.md): caching, routing, and output being priced above input.

</details>

---

### 8. Self-host or use an API? Give a decision rule, not a vibe.

`single-source`

<details>
<summary>Answer</summary>

1. Estimate monthly tokens, **input and output separately**, and peak concurrency. Output is usually priced several times higher than input on APIs, and reasoning models produce a lot of it.
2. Price that on the API.
3. Price the self-hosted version at a **realistic** utilization, not 100%. Include redundancy, and size the GPUs with the KV cache arithmetic from [question 5](#5-youre-asked-to-self-host-llama-3-70b-for-32-concurrent-users-at-8k-context-how-many-gpus).
4. Add engineering time.

Self-hosting usually wins only with **steady, high volume**, or when something other than price decides it: data that can't leave your boundary, a fine-tuned or custom model, or latency control an API won't give you. Low, spiky or uncertain volume favours APIs, because they turn utilization risk into someone else's problem.

The senior addition is to **evaluate quantization on your own task before pricing it in.** FP8 halving memory is the reason the numbers in question 5 work, but "nearly lossless" is a claim to test, not assume. See [model-evaluation.md](model-evaluation.md) for how quantization damage hides behind headline averages.

</details>
