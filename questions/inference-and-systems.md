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
