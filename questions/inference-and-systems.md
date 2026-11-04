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

### 5. You're asked to self-host Llama-3.1-70B for 32 concurrent users at 8K context. How many GPUs?

`single-source`

<details>
<summary>Answer</summary>

Size **two** things, not one. The mistake is sizing for the weights and then running out of memory on the KV cache.

**Weights** = parameters × bytes per parameter. That's 2 bytes at FP16/BF16, 1 at FP8/INT8, and about 0.5 at 4-bit (AWQ/GPTQ). So 70B is ~140 GB at FP16, ~70 GB at FP8 and ~35 GB at INT4.

**KV cache per token** = `2 (K and V) × layers × KV heads × head dim × bytes`. Llama-3.1-70B's config has 80 layers and 8 KV heads (grouped-query attention, against 64 query heads), with head dim derived as hidden size 8192 / 64 heads = 128:

```
2 × 80 × 8 × 128 × 2 bytes = 327,680 bytes ≈ 0.33 MB per token
32 users × 8,192 tokens   = 262,144 tokens
262,144 × 0.33 MB        ≈ 86 GB of KV cache
```

That's roughly the size of the FP8 weights again. Watch the units: GPU memory is quoted in GiB, so 86 GB of cache is 80 GiB, and the ~70.6B parameters at FP8 are ~66 GiB. Together that's ~146 GiB before activations and runtime overhead. That doesn't fit one H200 (141 GB), and it fills two 80 GB H100s (160 GiB) with almost no headroom. Your options:
- Quantize the KV cache too.
- Accept fewer concurrent full-length contexts. Paged attention only allocates what a request actually uses (PagedAttention, Kwon et al., arXiv 2309.06180).
- Add a GPU.

**The number to give:** two H100s is on the edge. Two H200s, or three H100s, is the safe answer, unless you shrink the cache.

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

That gives a back-of-envelope ceiling for a single stream: `tokens/s ≤ memory bandwidth / bytes of weights`. An H100 SXM has 3.35 TB/s (NVIDIA spec), so a 70 GB FP8 model tops out around **48 tokens/s for one user**, however many FLOPs sit idle. This is an upper bound; real numbers come in lower. The PagedAttention paper makes the same point from the serving side: throughput depends on batching, and the KV cache is what limits the batch (Kwon et al., arXiv 2309.06180).

The fix is **batching**. One read of the weights serves every sequence in the batch, so aggregate throughput rises almost linearly with batch size. That holds until you either become compute-bound or run out of KV cache memory, which ties this back to [question 5](#5-youre-asked-to-self-host-llama-31-70b-for-32-concurrent-users-at-8k-context-how-many-gpus). [Question 3](#3-which-inference-optimisations-would-you-reach-for-and-what-does-each-cost-you) tables these optimisations; this is the mechanism behind them. Continuous batching and paged KV caches in vLLM, SGLang and TensorRT-LLM exist to keep that batch full.

**Speculative decoding** attacks the same bottleneck from a different side. A small draft model proposes several tokens, and the large model verifies them in one pass. The original paper reports 2–3× speedups on T5-XXL with **identical outputs** — the same output distribution, not an approximation (Leviathan, Kalman & Matias, arXiv 2211.17192). The gain depends on how often the draft model is accepted, so measure it on your traffic.

The trade-off interviewers want named: **a strict latency SLA forces smaller batches, which raises cost per token.** Throughput and per-user latency pull against each other, and the SLA picks the point.

</details>

---

### 7. Turn a GPU bill into a cost per million tokens. What dominates it?

`foundational`

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

`foundational`

<details>
<summary>Answer</summary>

1. Estimate monthly tokens, **input and output separately**, and peak concurrency. Output is priced above input on APIs (see the illustrative prices in [question 2](#2-do-the-cost-arithmetic-out-loud-50000-daily-users-an-agent-averaging-12-llm-calls-per-session-4k-input-and-500-output-tokens-per-call)), and reasoning models produce a lot of it.
2. Price that on the API.
3. Price the self-hosted version at a **realistic** utilization, not 100%. Include redundancy, and size the GPUs with the KV cache arithmetic from [question 5](#5-youre-asked-to-self-host-llama-31-70b-for-32-concurrent-users-at-8k-context-how-many-gpus).
4. Add engineering time.

Self-hosting usually wins only with **steady, high volume**, or when something other than price decides it: data that can't leave your boundary, a fine-tuned or custom model, or latency control an API won't give you. Low, spiky or uncertain volume favours APIs, because they turn utilization risk into someone else's problem.

The senior addition is to **evaluate quantization on your own task before pricing it in.** FP8 halving memory is the reason the numbers in question 5 work, but "nearly lossless" is a claim to test, not assume. See [model-evaluation.md](model-evaluation.md) for how quantization damage hides behind headline averages.

</details>

---

### 9. Stream responses over SSE and handle disconnects.

`single-source`

<details>
<summary>Answer</summary>

Treat the stream as a delivery channel for a message your server owns. Generation and storage live on the server, and the browser connection is one viewer of it that can drop and come back. Most of the design follows from that choice. (Why streaming feels faster in the first place is covered in [cost-and-caching.md question 6](cost-and-caching.md#6-streaming-doesnt-reduce-total-cost-or-total-latency-why-do-teams-treat-it-as-a-latency-win-anyway); this question is about keeping it working.)

**The wire format.** An SSE response is `Content-Type: text/event-stream`, made of `field: value` lines. The WHATWG HTML spec defines four fields: `data`, `event`, `id` and `retry`. A blank line dispatches the event, and a line starting with `:` is a comment the client ignores. If the server sets `id`, the browser's `EventSource` remembers it and sends it back in a `Last-Event-ID` request header when it reconnects. `retry` sets the reconnect delay in milliseconds. The spec also says when the client gives up for good: a non-200 status or the wrong content type closes the connection with no retry, and a 204 tells it to stop reconnecting. A plain network drop or a normally ended stream makes it reconnect (WHATWG HTML, "Server-sent events", fetched 2026-11-02).

```
id: msg_42:17
event: delta
data: {"text": " world"}

: keepalive
```

**Proxies and timeouts.** The most common "streaming doesn't work" bug is a proxy buffering the whole response. nginx has `proxy_buffering on` by default. Turning it off passes the response to the client "synchronously, immediately as it is received", and the upstream can do the same per response by sending `X-Accel-Buffering: no` (nginx `ngx_http_proxy_module` docs, fetched 2026-11-02). Compression middleware can buffer too, so check that as well. Idle timeouts are the second problem. nginx's `proxy_read_timeout` defaults to 60s between reads, and cloud load balancers have their own idle timeouts. A model thinking for a long time, or a slow tool call, produces silence, and the connection gets cut. The spec suggests sending a comment line about every 15 seconds for this reason. Anthropic's stream also contains `ping` events, but those keep your connection to Anthropic alive, not your user's connection to you, so you still need your own heartbeat.

**What happens upstream when the user leaves.** This is where people guess, so be precise. Anthropic's streaming docs and OpenAI's streaming guide (both fetched 2026-11-02) say nothing about whether closing the provider connection stops generation, or whether you're billed for tokens generated after you stop reading. Neither page documents it. The safe assumption for cost planning is that you may pay for the full generation. That leaves you with a real decision:

- Abort the upstream request when the client disconnects. This saves tokens if the provider stops, but a user who refreshes or loses Wi-Fi gets nothing to resume.
- Keep generating to completion regardless. The answer is ready when they come back, at the price of paying for answers nobody reads.

A common middle ground is to keep going through short disconnects (a grace period of some seconds) and abort if nobody reattaches. An explicit Stop button is a different signal from a dropped connection, and it should always abort.

**Resumable streams.** Decouple generation from the HTTP request. A worker consumes the provider stream and appends each chunk to a store keyed by message id (Redis streams or a list work well), with a sequence number. The SSE handler reads from that store, and it uses `msg_id:seq` as the SSE `id`. On reconnect, `Last-Event-ID` tells you where to restart: replay everything after that sequence, then tail live. That also gives you multi-tab viewing for free. Set a TTL on the buffer and write the final message to the database when generation ends. Note that native `EventSource` only does GET and can't set custom headers, so many chat apps use `fetch` with a streaming body reader instead. If you do that, you implement the reconnect and last-id logic yourself.

**Cancellation and what you save.** Decide what ends up in chat history when a message stops early, because the next turn will send it back to the model as context. Save the partial text with a status (`complete`, `stopped_by_user`, `interrupted`, `error`) and the stop reason. A partial `tool_use` block is a trap: the input JSON arrives as partial strings and is only valid at `content_block_stop`, so never execute or store a half-built tool call as if it were real. Drop it or mark it incomplete. Anthropic's docs say the same about recovery: tool use and thinking blocks can't be partially recovered, and you resume from the most recent text block.

**Errors mid-stream.** The HTTP status is already 200 by the time the first token goes out, so a failure later has to arrive as an event. Anthropic sends an `error` event inside the stream, for example `overloaded_error`, which would be an HTTP 529 in a non-streaming call. Its docs describe a recovery pattern: keep the partial response, then send a continuation request (as a prefilled assistant message on Claude 4.5 and earlier, or as a user message asking it to continue on 4.6 and later). OpenAI's guide lists an `error` event type too. Your own SSE stream should forward these as a typed event the UI can render, with the partial text kept and a retry button, and not just close the connection, since `EventSource` would read a closed connection as a cue to reconnect.

**SSE or WebSockets?** For token streaming, SSE is usually the better default. It's plain HTTP, it goes through most proxies once buffering is off, and reconnection with a last id is in the spec. The traffic is one-way anyway, since the user's next message can be a normal POST. Reach for WebSockets when you need real two-way traffic on one connection: voice, collaborative editing, or interrupting the model mid-generation at low latency. You then build your own reconnect, resume and heartbeat logic, and some corporate proxies handle WebSocket upgrades badly.

</details>

---

### 10. Cut first-token latency for a chat app.

`single-source`

<details>
<summary>Answer</summary>

Start by finding out where the time goes, because time to first token (TTFT) is a sum of several waits and only one of them is the model. A user's TTFT is roughly:

```
TTFT = network + TLS handshake
     + your gateway (auth, rate limits, moderation)
     + work before generation (retrieval, tool calls, history loading)
     + queueing at the provider or your inference server
     + prefill (processing the prompt)
     + time to the first visible token (thinking, if the model reasons first)
```

Instrument each segment as a span (see [question 4](#4-how-do-you-make-a-genai-system-observable-in-production)) before changing anything. In many chat apps the biggest term turns out to be the app's own pre-generation work, which no model change will fix.

**Measure from both ends, at p50 and p95.** Server-side spans tell you which segment is slow. The client measurement, from the user pressing send to the first rendered token, tells you what the user actually feels, and it includes things the server never sees: DNS, TLS, mobile networks and proxy buffering. Watch p95 as well as p50. Queueing and retrieval tend to hide in the tail, and a median that looks fine can sit beside a p95 users complain about. Split the numbers by prompt length and by cache hit or miss, or a mix shift will look like a regression.

**Then the levers, roughly cheapest first.**

- Do less before generation. Run independent retrieval and tool calls in parallel, not one after another. Skip retrieval when a cheap check says the turn doesn't need it ("thanks", a follow-up on the previous answer). Shorter prompts also mean less prefill. The trade-off is quality: a skipped retrieval that was needed gives a confident wrong answer, so evaluate the skip rule.
- Cache the prefix. Prompt caching lets the provider skip prefill on a stable prefix, which helps TTFT as well as cost. The layout rules and the ways teams break their own cache are in [cost-and-caching.md question 3](cost-and-caching.md#3-what-makes-a-prompt-cache-friendly-and-how-do-people-accidentally-defeat-their-own-cache).
- Use a smaller or faster model, or route. Anthropic's latency guide (fetched 2026-11-02) puts choosing the model first among its levers and points speed-critical apps at its fastest tier, with effort as the control. Routing easy turns to the small model keeps quality where it matters; routing has its own trade-offs, covered in [cost-and-caching.md question 4](cost-and-caching.md#4-when-do-you-reach-for-model-routing-or-cascades-instead-of-or-alongside-caching).
- Cut network time. Call the provider region nearest your servers, keep HTTP connections alive and pooled so you don't pay a TLS handshake per request, and terminate the user's connection close to them.
- Stream. Streaming doesn't make the model faster, but it puts the first token on screen as soon as it exists instead of after the whole answer. Why that counts as a latency win is in [cost-and-caching.md question 6](cost-and-caching.md#6-streaming-doesnt-reduce-total-cost-or-total-latency-why-do-teams-treat-it-as-a-latency-win-anyway), and making it survive proxies is [question 9](#9-stream-responses-over-sse-and-handle-disconnects).

**Reasoning models move the goalposts.** With extended thinking on, the model produces thinking tokens before the answer, so the first *visible* token can arrive long after the first generated one. Lower the effort or thinking budget for chat turns that don't need it, route those turns to a non-reasoning setting, or show the thinking stream (or a status line) so the wait isn't a blank screen. Measure the time to first answer token separately from TTFT.

**If you self-host, the server has its own levers.** Long prefills from new requests stall the decodes of everyone already streaming, and queued requests wait behind them. Three techniques address that:

- Chunked prefill splits a long prompt into chunks and mixes them into batches with ongoing decodes, so new requests don't freeze everyone else. Sarathi-Serve reports up to 2.6x higher serving capacity than vLLM for Mistral-7B on one A100 under tail-latency constraints (Agrawal et al., arXiv 2403.02310).
- Prefix caching keeps the KV cache for shared prefixes (system prompt, tool definitions) so repeated prefill is skipped. It's the self-hosted version of provider prompt caching.
- Prefill/decode disaggregation runs the two phases on separate GPUs so they stop interfering, and tunes each for its own target (TTFT for prefill, time per output token for decode). DistServe reports up to 7.4x more requests served, or a 12.6x tighter SLO, while keeping over 90% of requests within latency limits (Zhong et al., arXiv 2401.09670). The cost is moving the KV cache between GPUs, which needs fast interconnect.

These are paper results on the authors' setups, so treat them as direction and benchmark on your own traffic. The trade-off underneath all of them is the one from [question 6](#6-why-is-single-user-decode-so-slow-on-a-gpu-that-costs-this-much-and-what-fixes-it): bigger batches lower cost and raise queueing, so a TTFT target at p95 limits how hard you can batch.

I'd answer in that order: measure, remove pre-generation work, cache, then pick the model, and only then touch serving internals. Sources: Anthropic "Reducing latency" docs; arXiv 2401.09670 and 2403.02310 abstracts, all fetched 2026-11-02.

</details>

---

### 11. Your LLM pipeline takes 8 seconds end to end. Make it faster.

`single-source`

<details>
<summary>Answer</summary>

Cut output tokens and sequential calls first, because those are where an LLM pipeline spends its time. OpenAI's latency guide gives two rules of thumb that set the priorities: "cutting 50% of your output tokens may cut ~50% of your latency", while halving the prompt may only improve latency by 1 to 5% (OpenAI, "Latency optimization", fetched 2026-11-04). Those are OpenAI's figures for its own API and its own example, so treat them as direction and check them on your traffic. Before touching anything, measure.

**Measure per step.** "8 seconds" is a sum, and the fix depends on which term is large. Put a span around every model call, retrieval, tool call and post-processing step (see [question 4](#4-how-do-you-make-a-genai-system-observable-in-production)), and record input tokens, output tokens and wall time for each. Here is an illustrative breakdown, invented for this answer and not measured anywhere:

```
rewrite the user query     1.2 s   (LLM call, 40 output tokens)
retrieval                  0.4 s
route / classify           1.0 s   (LLM call, 15 output tokens)
main answer                4.6 s   (LLM call, 350 output tokens)
validate and format        0.8 s   (LLM call, 120 output tokens)
                           -----
total                      8.0 s
```

Even in this made-up case, two of the four model calls produce almost nothing yet cost over two seconds between them. That is mostly per-request overhead and queueing, which shrinking the prompt would barely touch. If your real trace looks like this, the sequence of calls is the problem. If one call dominates, the output length of that call is.

**Why trimming the prompt rarely pays.** The model reads the prompt in one parallel pass (prefill) and then writes the answer one token at a time, so each output token costs far more wall time than each input token. That is why the 1 to 5% figure is so small. The guide says input reduction is worth pursuing mainly for very large contexts. So if your prompt is a 100K-token document dump, shortening it matters; if it is a 2K system prompt, leave it alone. For the cases where context really is large, put static content first and dynamic content (retrieved passages, history) later so the provider's prefix cache can skip the repeated part. The layout rules are in [cost-and-caching.md question 3](cost-and-caching.md#3-what-makes-a-prompt-cache-friendly-and-how-do-people-accidentally-defeat-their-own-cache).

**Generate fewer tokens.** Ask for brevity, show short examples, set a stop sequence or a max length, and shrink structured-output syntax. The guide's worked example shortened JSON field names and moved explanations out of the output, which removed 19 output tokens and could save up to a second on GPT-4 (a model generation that is now old, so the absolute saving will differ today). The cost is quality: a model told to be terse sometimes drops the reasoning that made its answer right, so rerun your evaluation set after each cut.

**Make fewer requests by merging steps.** In the sketch above, query rewriting and routing both read the same user message and could be one call. Ask for an enumerated list of steps and a JSON object with named fields, such as `{"query": "...", "needs_retrieval": true}`, so the next stage can parse each result. The guide's example did exactly this and then moved the merged step to a smaller fine-tuned model. The downsides are real:

- A merged prompt is harder to get right, and one weak instruction can degrade the other task. Evaluate each field separately against the old two-call version.
- You lose independent retries, caching and per-step model choice. A failure in one field forces you to rerun both.
- The combined prompt can be longer and its output bigger, so part of the saving goes back. The guide itself frames splitting versus merging as a trade-off to settle by testing.

**Run independent steps in parallel.** The guide's line is that two shirts take as long to dry as one. If retrieval and the routing check don't depend on each other, start both and wait for the slower. Latency becomes the longest branch instead of the total. Parallel calls don't reduce tokens, so cost stays the same, but you will hit rate limits sooner and have to handle partial failure.

**Speculative execution** is for steps that really are sequential but where one outcome is likely. The guide's example is a moderation check: start the main generation at the same time as the check, and cancel the generation if the check says no. When the guess is right, the check adds roughly nothing to latency. When it is wrong, you have paid for tokens you throw away, so the trick only earns its keep when the likely branch is common, say 95% of traffic, and the wasted branch is cheap. Work out the break-even from your own hit rate and token prices.

**Predicted outputs** apply when most of the answer is already known, as in editing a file where only a few lines change. You send the expected text in the `prediction` parameter of Chat Completions, and the model can verify matching tokens faster than it can generate them. OpenAI's predicted outputs page (fetched 2026-11-04) lists the limits: it names the GPT-4o and GPT-4.1 families (including mini and nano) as supported, and several parameters don't work with it, including `tools`, `logprobs`, `n` above 1, `max_completion_tokens`, audio and positive presence or frequency penalties. On cost, prediction tokens that don't end up in the final answer are billed at completion-token rates, and the page warns that it "can introduce higher costs". Usage reports accepted and rejected prediction tokens separately, so track the rejected share. The page gives no speedup figure, so benchmark it. A poor prediction costs money and gains no latency, which makes this a good fit for code or document rewrites and a bad one for open-ended chat.

**Use a smaller model where the task allows.** Token speed depends mainly on model size, and the guide notes that a smaller model can match a larger one on a narrow task when it gets a more detailed prompt, few-shot examples or fine-tuning. Route the cheap steps, such as classification and extraction, to the small model, and keep the large one for the final answer. You pay in evaluation work and in a second model to maintain. Routing trade-offs are in [cost-and-caching.md question 4](cost-and-caching.md#4-when-do-you-reach-for-model-routing-or-cascades-instead-of-or-alongside-caching).

**Take the LLM out of a step.** The guide's last principle is not to default to a model. Fixed confirmations and refusals can be hard-coded, answers for a small set of inputs can be precomputed, and a lookup, a regex or a cache can replace a call. In the sketch, "validate and format" at 0.8 s is a candidate: a JSON schema check and a template do the same job in milliseconds, and the model call can stay only for the rare failures.

**What I'd do with the sketch.** Merge rewrite and routing into one small-model call, run retrieval alongside it where the data allows, cap and tighten the main answer's length, and replace the validator with code. If the answer still takes several seconds, what remains is the user's perceived wait. Streaming addresses that and is covered in [question 10](#10-cut-first-token-latency-for-a-chat-app) and [question 9](#9-stream-responses-over-sse-and-handle-disconnects); [cost-and-caching.md question 6](cost-and-caching.md#6-streaming-doesnt-reduce-total-cost-or-total-latency-why-do-teams-treat-it-as-a-latency-win-anyway) explains why it counts. The guide's own caution applies to all of this: its principles conflict (fewer requests versus faster tokens, for one), so change one thing at a time and measure on production examples.

Sources: OpenAI "Latency optimization" guide (developers.openai.com/api/docs/guides/latency-optimization) and OpenAI "Predicted Outputs" guide (developers.openai.com/api/docs/guides/predicted-outputs), both fetched 2026-11-04. The 8-second breakdown, the 95% threshold and the sketch's fixes are illustrative reasoning, not source data.

</details>

---

### 12. Design an end-to-end batching system for LLM queries.

`single-source`

<details>
<summary>Answer</summary>

Build a durable job system with one row per item, and treat each provider batch as a disposable transport for those rows. A batch can expire, partly fail or return results out of order, and none of that should lose an item or pay for it twice.

First ask which "batching" they mean. Offline batch APIs take a set of requests and return results asynchronously at a discount. Dynamic or continuous batching groups live requests on a self-hosted GPU to raise throughput, which is covered in [question 3](#3-which-inference-optimisations-would-you-reach-for-and-what-does-each-cost-you) and [question 6](#6-why-is-single-user-decode-so-slow-on-a-gpu-that-costs-this-much-and-what-fixes-it). The rest of this answer is the offline system.

**What the providers document** (all fetched 2026-11-04):

| | OpenAI Batch API | Anthropic Message Batches |
|---|---|---|
| Price | 50% discount against synchronous APIs | All usage at 50% of standard prices |
| Window | `24h` is the only option | Expires if unfinished after 24 hours; most finish within 1 hour |
| Batch size | 50,000 requests or a 200 MB input file | 100,000 requests or 256 MB, whichever comes first |
| Queue limit | Queued prompt tokens, per model | Queued batch requests, shared across models (500,000 on the Scale tier) |
| Result order | May not match input order | Any order |
| Result retention | Output file deleted 30 days after completion | 29 days from batch creation |
| Interactive limits | Separate pool | Batch usage doesn't affect Messages API limits |

**Intake and idempotency.** A client submits a job with a model, prompt version, parameters and deadline, and the items go into your own store before anything reaches a provider. I'd keep tables for jobs, items (`pending`, `in_batch`, `succeeded`, `retry_wait`, `quarantined`) and provider batches, so a crashed worker can restart from the rows. A client-supplied job ID stops a retried submission creating a second job. A content hash over model, prompt version, parameters and input lets you reuse earlier results, which suits extraction and is wrong when you want several samples, so make it a per-job option.

Neither provider's batch guide or batch-creation API reference documents an idempotency key for creating a batch. The risky moment is a crash after the provider accepted a batch and before you saved its ID. Record the chunk as `submitting` first, tag the batch with your chunk ID where you can (OpenAI's batch object takes a `metadata` map of up to 16 pairs; Anthropic's has no batch-level metadata, only the per-request Messages `metadata.user_id`), and list recent batches on recovery before resubmitting. A duplicate batch then costs money but can't corrupt state, because the first terminal result per item wins.

**Chunking and rate limits.** Split by whichever limit bites first, which is often bytes. At an illustrative 5 KB per request, 256 MB holds about 51,000 requests and 200 MB about 40,000. Smaller chunks give finer progress and a smaller blast radius, at the price of more batches to track. Then admit chunks against the queue limit. With illustrative numbers, a 2M-item job on Anthropic's Scale tier fits five full batches in the queue at once, and the next chunk waits for earlier ones to drain. A fair scheduler stops one backfill from holding the whole queue.

**Partial failure.** Decide per item, since a batch's status says little. Anthropic returns `succeeded`, `errored`, `canceled` or `expired` for each request, and OpenAI puts failures in a separate error file, with `batch_expired` for expired requests. My policy:

- Server errors go back into a later chunk, with backoff and a capped attempt count.
- Invalid requests aren't retried, because they'll fail the same way.
- Expired items are a capacity problem, so they requeue without using an attempt, if the deadline allows another 24-hour window.
- An item that exhausts its attempts is a poison item. Quarantine it with the error attached and finish the job as `completed_with_errors`, so one record can't hold up the rest.

Check successes too. A result truncated at `max_tokens` or failing your schema is billed and useless. Anthropic reports `params` validation errors only when the whole batch has ended and suggests dry-running the request shape on the Messages API, so one synchronous call at intake can save a day of waiting.

**Reconciliation.** Results are unordered, so `custom_id` is the only join. Use something like `<item id>_a<attempt>`, keep customer data out of it, and respect Anthropic's `^[a-zA-Z0-9_-]{1,64}$`. Copy results to your own storage as soon as a batch ends, because of the retention windows. Ingest with an upsert so replaying a file is harmless, then confirm every submitted ID came back, and requeue and alert on any that didn't. Cancelling is asynchronous on both, so keep ingesting afterwards: you may have paid for those results.

**Cost and caching.** Estimate and enforce the budget before submission, because Anthropic says batches may go slightly over a Workspace spend limit. Record actual usage per item and roll it up as in [cost-and-caching.md question 5](cost-and-caching.md#5-how-do-you-attribute-llm-spend-so-you-can-actually-answer-why-did-the-bill-go-up). Anthropic doesn't bill `errored`, `canceled` or `expired` requests. OpenAI charges for requests completed in an expired batch.

Anthropic says the caching and batch discounts can stack, but hits are best-effort because requests run concurrently and in any order. It quotes typical hit rates of 30% to 98%, suggests the 1-hour cache duration for shared context, and doesn't support `max_tokens: 0` pre-warming inside a batch. OpenAI's batch and caching guides don't discuss the interaction, and its pricing page lists a Batch cached input price without stating the rule. I'd budget at the uncached rate, count hits as upside, and measure cached tokens per chunk.

**Progress and priority.** Report from your item table: counts per state, spend so far and projected finish, since a job can span several batches and attempts. Plan around the 24-hour commitment and ignore "most finish within 1 hour" for ETAs. Provider batches don't eat live quota, so the danger is your own escape hatch: rescuing stragglers through the synchronous API costs full price and draws on your users' limits, so give that path its own low-ceiling token bucket. A job due within a day shouldn't rely on batch at all. On your own GPUs there's no separate pool, so run batch at the lowest priority and preempt it when live latency slips.

**Evals and pinning.** Pin a dated model version where one exists and store the configuration on the job, so retries over several days can't mix outputs from two versions unnoticed. Drift and retirements are in [llmops.md question 3](llmops.md#3-your-provider-may-change-a-model-behind-the-same-id-how-would-you-detect-a-silent-regression) and [question 5](llmops.md#5-your-production-model-has-a-retirement-date-walk-through-the-migration-playbook). Score a canary first (an illustrative 200 stratified items) and release the rest only if it passes. Sample the finished output as well, oversampling retried items, and hold results from downstream systems until that passes.

All this machinery pays off on large, repeated, deadline-tolerant work such as backfills and eval runs. For a small job, a synchronous worker with a rate limiter may cost less overall despite the higher token price.

The state model, retry policy, routing rules and illustrative numbers are my own design reasoning. Sources: OpenAI Batch API guide and batch-creation API reference, prompt caching and rate limits guides and pricing page; Anthropic batch processing guide, batch-creation API reference and rate limits docs, all fetched 2026-11-04.

</details>
