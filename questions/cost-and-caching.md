# Cost and caching

The cost-arithmetic exercise — do the token math for a 50K-user agent workload out loud — already lives in [inference-and-systems.md](inference-and-systems.md#2-do-the-cost-arithmetic-out-loud-50000-daily-users-an-agent-averaging-12-llm-calls-per-session-4k-input-and-500-output-tokens-per-call), along with the optimisation trade-off table. This file picks up where that one stops: the two caching mechanisms an interviewer expects you to tell apart, how to design a cache for a real chatbot, why the same context window can be cache-friendly or cache-hostile depending on how you write it, when routing beats caching, and how teams actually verify the savings they claim.

---

### 1. Prompt caching and semantic caching sound similar. What's actually different?

`verified 3-0`

<details>
<summary>Answer</summary>

They solve different problems and neither substitutes for the other.

**Prompt caching** is exact-prefix reuse at the provider level. You send the same leading tokens (system prompt, tool definitions, a long document) across calls, the provider recognises the identical prefix and skips reprocessing it — cheaper and faster, but only for byte-identical prefixes. It buys you nothing on the parts of the prompt that change every turn.

**Semantic caching** sits in front of the model entirely. You embed the incoming query, compare it against previously answered queries by similarity, and return a stored response on a close-enough match — skipping the LLM call altogether. It works on *different* requests that mean the same thing ("How do I reset my password?" vs "I forgot my password, help"), which prompt caching cannot touch since the token sequences aren't identical.

The trade-off that matters in an interview: prompt caching is safe — you get back exactly what a fresh call would have produced, just cheaper. Semantic caching is a bet — a similarity match can return a *wrong* answer to a query that's close but not equivalent ("cancel my subscription" vs "pause my subscription"), so it needs a similarity threshold, a staleness policy, and usually a scope restriction to low-stakes, high-repetition query types.

Primary evidence for the sizes of each win: prompt caching cut agent-task API costs by **41–80%** and time-to-first-token by **13–31%** across OpenAI, Anthropic and Google on long-horizon tasks (arXiv 2601.06007, DeepResearch Bench, 500+ sessions — one benchmark, treat the range as suggestive not universal). Semantic caching cut API calls by up to **68.8%** in its best category in a Redis-backed implementation (arXiv 2411.05276, preprint, not peer-reviewed). Neither number transfers automatically to your workload — both papers measured narrow benchmarks.

</details>

---

### 2. Design a cache for a customer-support chatbot handling 50,000 queries a day. Walk through it.

`single-source`

<details>
<summary>Answer</summary>

The strong answer is layered, not a single mechanism — and says which layer does the most work.

**Layer 1 — prompt caching on the stable prefix.** System prompt, tool/function definitions, product docs or policy text injected into every call: put these first in the prompt and keep them byte-identical turn to turn. This is the highest-leverage, lowest-risk layer because it requires no similarity judgment — it's exact-match, so it can never return a wrong answer.

**Layer 2 — semantic cache on the query itself.** Support chatbots have a long tail of near-duplicate questions ("how do I reset my password", "forgot password help"). Embed the incoming query, check against a vector store of previously answered queries, and serve the stored response above a similarity threshold. Scope this to FAQ-shaped queries with stable, low-stakes answers — not account-specific or transactional queries, where a near-miss match is a real failure mode, not just a missed saving.

**Layer 3 — model routing.** Classify query difficulty (a cheap classifier or the first few tokens of a small model's response) and route simple queries to a smaller/cheaper model, reserving the frontier model for queries that need reasoning, ambiguity resolution, or tool orchestration.

**Layer 4 — cost attribution from day one.** Tag every call with feature, user segment, model, and token counts so you can find out *which* fraction of the 50K/day is actually cacheable before you over-invest in infrastructure for a workload that turns out to be mostly unique queries.

The trap worth naming out loud: **build in this order.** Prompt caching first because it's free correctness-wise; semantic caching second because it needs guardrails; routing third because it needs a working difficulty classifier. Interviewers are listening for whether you sequence by risk, not just list all four.

</details>

---

### 3. What makes a prompt "cache-friendly," and how do people accidentally defeat their own cache?

`verified 3-0`

<details>
<summary>Answer</summary>

Prompt caching works on **exact-prefix identity**, so the practical rule is: put everything stable first, everything variable last.

**Layout that works:** system prompt → tool/function definitions → static reference material (docs, schemas) → few-shot examples → *then* the turn-specific user content and any content injected mid-conversation. Every token after the first point of divergence from a previous call is uncached, so pushing the volatile part as late as possible maximises the reused prefix.

**Cache-busting mistakes to name:**
- A timestamp, request ID, or "current date" string inserted into the system prompt — it changes every call and invalidates the entire downstream prefix even though nothing else moved.
- Reordering tool definitions or injecting them dynamically based on which tools are "relevant" this turn — looks like an optimisation, actually destroys the one thing making the prefix reusable.
- Full-context caching (cache the whole conversation history every turn) instead of caching selected stable blocks. arXiv 2601.06007 found that caching selected blocks (system prompt cached, dynamic content and tool results left out) beat caching full context — full-context caching sometimes *increased* latency, likely because a large invalidated cache still has to be rewritten.

</details>

`single-source` — the pricing figures below are current vendor-docs numbers, not part of the 3-0 verification above.

<details>
<summary>On the numbers</summary>

Current published pricing and minimums (fetched 2026-09-25) — Anthropic's platform.claude.com prompt-caching docs list cache reads at 0.1x base input price for most models (with cheaper 0.025x–0.05x rates for some), cache writes at 1.25x (5-minute TTL) or 2x (1-hour TTL), and minimum cacheable prefixes ranging 512–4,096 tokens by model. OpenAI's developers.openai.com prompt-caching guide states cached input tokens are discounted "up to 90%" (0.1x rate for GPT-5.6+) with a 1,024-token minimum for GPT-5.6 and later, varying for earlier models. These numbers move — a Neuraltrust and an Introl blog post citing a "50% off" automatic-discount claim and a ~1.4-reads-per-write break-even both failed independent verification in preparing this file, not because those figures were confirmed wrong but because they could not be confirmed from the blogs cited (see [reference/refuted.md](../reference/refuted.md)) — confirm against the current doc page before quoting a number in an interview, and say the date you checked.

</details>

---

### 4. When do you reach for model routing or cascades instead of, or alongside, caching?

`single-source`

<details>
<summary>Answer</summary>

Caching saves money when the *same* work recurs. Routing saves money when the work varies in difficulty — which is most workloads, and it composes with caching rather than competing with it.

**Simple routing:** classify the incoming request (rules, a cheap classifier, or the small model's own confidence) and send it to the cheapest model that can handle it, escalating only on failure or low confidence. Google's writeup of engineering patterns from their AI agents challenge is worth citing here: one team's **tiered routing caught 40%+ of incoming messages with a zero-token regex pass** before any model call — the cheapest possible tier is sometimes no model at all (developers.googleblog.com, fetched 2026-09-25). A related pattern in the same writeup, "same-bar fallback," is worth naming alongside it: a single `validate_clinical_response()` function that both the primary ("Pro") and fallback ("Flash") paths are forced to call before either result can leave the agent, so falling back to a cheaper model on failure can't silently degrade output quality just because it saved tokens.

A production example of the same idea at the tool-call level: Spotify's Portal routes bulk file reads to a cheaper worker model (Gemini 2.5 Flash) instead of the frontier model doing the reading itself, and reports **mean token savings of around 90% on those bulk reads**. The post is also honest about the failure mode routing introduces: the cheaper worker missed a subtle thread-safety bug that the frontier model caught once given the right context, so routing bulk I/O away from the expensive model doesn't remove the need for the expensive model to look at anything that actually requires judgment (engineering.atspotify.com, fetched 2026-09-25).

**Cascades** go further: start with the cheap model, and only invoke the expensive model when the cheap one signals uncertainty or its output fails a check. This differs from static routing because the decision to escalate is made *after* seeing the cheap model's attempt, not before.

**Lead-and-executor fusion** is a variant worth naming for a senior conversation: **Devin Fusion (Cognition)** pairs a frontier "lead" model that plans and reviews with a cheaper "sidekick" that executes, exchanging structured briefs and results rather than full transcripts (cognition.com, fetched 2026-09-25, claims 36–39% cost savings from the vendor). The interesting design point isn't the savings figure — it's that the lead can **take control back mid-task** when the sidekick is out of its depth, which static routing can't do because it commits to a model before the task's real difficulty is known. That's the general argument for cascades over routing: routing bets up front, cascades and fusion bet incrementally.

The trade to name honestly: every extra hop (classify, attempt cheap, verify, escalate) adds latency and engineering surface. Routing only pays for itself when the cheap tier's hit rate is high enough that the average case wins by more than the worst case loses.

</details>

---

### 5. How do you attribute LLM spend so you can actually answer "why did the bill go up"?

`single-source`

<details>
<summary>Answer</summary>

The default failure mode is a single opaque number from the provider invoice. The fix is tagging every call at the point of the request, not reconstructing it after the fact from logs.

Tag each call with: **feature or product surface** (which part of the app triggered this), **user or account segment**, **model and version**, **input/output token counts**, and **cache status** (hit, miss, write) if you're running prompt or semantic caching. That last field matters specifically because a cache hit and a cache miss have wildly different costs for what looks like the same logged "call" — without it, cost-per-feature numbers are noise.

Push these into the same trace as your OpenTelemetry GenAI spans (see [inference-and-systems.md question 4](inference-and-systems.md#4-how-do-you-make-a-genai-system-observable-in-production)) so cost attribution and latency/quality observability share one pipeline instead of two.

Worth raising unprompted: **a reported cache hit is not proof that work was actually skipped.** A verification-focused writeup on this argues a cache event only counts as real evidence when several things line up together: an independent prediction of what should be cacheable, the runtime's own attestation, confirmation that the system actually skipped reprocessing those tokens, identical output between cached and fresh runs, a correctness check on the result, and everything tied together with something like a hash or checksum so the pieces can't drift apart (siddhantkhare.com, "A cache hit is not proof", fetched 2026-09-25). For an interview answer this compresses to: don't take a cache-hit-rate dashboard at face value when the dollar savings matter — spot-check that a hit actually corresponds to skipped computation and an identical answer.

A second reason attribution has to be per-call rather than per-model: routing the same logical "model" through a broker like OpenRouter can mean different backend providers serve the request with different serving stacks, so cost *and* behaviour can vary under one nominal model ID (simonw.substack.com, fetched 2026-09-25). If you attribute cost to "gpt-x via OpenRouter" as one bucket, you can miss that one underlying provider is both slower and pricier than another behind the same endpoint.

</details>

---

### 6. Streaming doesn't reduce total cost or total latency. Why do teams treat it as a latency win anyway?

`foundational`

<details>
<summary>Answer</summary>

Because the metric users feel is **perceived** latency, not total generation time, and streaming moves the number that governs perception without moving the number that governs cost.

Total tokens generated, and total cost, are unchanged by streaming — you still pay for and wait for every output token. What changes is **time to first token (TTFT)**: the user sees the response start appearing almost immediately instead of staring at a blank state until the full answer is ready. For anything longer than a short answer, TTFT dominates the user's sense of "is this fast," which is why it's tracked as a first-class metric alongside tokens/second and total latency (see [inference-and-systems.md question 1](inference-and-systems.md#1-design-a-claude-chat-service-where-does-this-interview-actually-go)).

This is also where streaming and caching interact rather than compete: prompt caching directly improves TTFT (arXiv 2601.06007 measured 13–31% TTFT improvement alongside the cost win), so a cached, streamed response gets both a real latency win on the first token and a perceived-latency win on everything after it. The honest caveat to volunteer: streaming a bad answer faster is not a win — it just gets the user to the wrong answer sooner, which is an argument for keeping cheap correctness checks (schema validation, a fast classifier) ahead of the stream rather than only at the end.

</details>

---

### 7. Your CI shows a cache hit rate that looks too good. How do you find out if it's real?

`single-source`

<details>
<summary>Answer</summary>

Treat a reported hit rate the way you'd treat any other self-reported metric from a system with an incentive to look efficient: verify it against something the caching layer doesn't control.

The concrete checklist, adapted from the "cache hit is not proof" framing above: (1) have an independent expectation of what *should* be cacheable, based on which tokens are actually identical across calls, not the runtime's word for it; (2) get the runtime's own attestation of what it cached; (3) confirm the system's observed work — tokens actually processed — is smaller on a hit than a miss, not just that a flag says "hit"; (4) check the cached and non-cached outputs are token-identical, since a "hit" that returns a different answer isn't a hit, it's a bug; (5) run your correctness evaluator over both paths — a hit that returns wrong output has helped no one; (6) where it matters enough to justify the effort, bind the whole chain together with a hash or checksum so the claim can be audited later rather than trusted once.

This matters in production for a mundane reason: things that look like caching infrastructure can quietly stop caching without breaking anything visible. Anthropic's own Claude Code changelog has shipped fixes for exactly this shape of bug — for example an entry (v2.1.280, fetched 2026-09-25) fixing a model switch made mid-task from a host app causing a prompt-cache miss on the very next prompt. Nothing errored; the cache silently stopped paying off. The same release also fixed resumed fork subagents rebuilding their tool list, which is arguably the better example to reach for in an interview: reordering tool definitions is exactly the cache-busting mistake named in [context-engineering.md question 6](context-engineering.md#6-how-do-you-keep-an-agents-prompts-cache-friendly), and here it happened silently as a side effect of resuming a subagent rather than from anyone editing a prompt. If your dashboard reports a stable hit rate, that's a reason to check harder, not a reason to stop checking — dashboards report what the instrumentation was told to report, not what happened underneath it.

</details>
