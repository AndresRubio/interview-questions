# LLMOps and production observability

The basics of observability are already in [inference-and-systems.md](inference-and-systems.md#4-how-do-you-make-a-genai-system-observable-in-production): trace calls as spans, record tokens and TTFT, and treat prompt changes as code changes. Offline evaluation (judge validation, labels, reading traces) is in [evaluation.md](evaluation.md), and confidence thresholds that drift once deployed are in [uncertainty-quantification.md question 6](uncertainty-quantification.md#6-you-have-a-confidence-score-how-do-you-know-its-any-good). This file covers what happens after launch: tracing a multi-step agent in a way that survives a vendor change, evaluating live traffic, catching a model that changes under a fixed ID, rolling out prompts safely, migrating off a model before its retirement date, running an incident, and setting SLOs for a system whose failures are often quiet.

---

### 1. How would you trace an agent run end to end, and what is the actual status of the OpenTelemetry GenAI conventions?

`single-source`

<details>
<summary>Answer</summary>

Treat each user request as one trace and make the agent's structure visible in the span tree. Use one `invoke_agent` span for the run, a child span for each model call, and an `execute_tool` span for each tool call. Put `gen_ai.conversation.id` on the spans so that separate turns can be joined back into a session. These operation names come from the OpenTelemetry GenAI agent-span conventions. There are also agent-level metrics such as `gen_ai.invoke_agent.duration`, `gen_ai.invoke_agent.tool_calls` and `gen_ai.execute_tool.duration`.

Senior candidates know three facts that others usually don't:

- **The spec is still in Development status, and it has moved.** As of 2026-09-25, the GenAI pages on opentelemetry.io are marked "Moved". The conventions now live in a separate repository, `open-telemetry/semantic-conventions-genai`, and the model spans, agent spans and metrics pages there are all marked **Development**. That means attribute names can still change. Keep instrumentation behind a thin wrapper of your own, and pin the semconv version your collector and backend expect. Don't scatter attribute names through application code.
- **Prompt and completion content is opt-in, and that's deliberate.** `gen_ai.input.messages`, `gen_ai.output.messages`, `gen_ai.system_instructions` and `gen_ai.tool.definitions` are all `Opt-In`. The spec warns that the message attributes are likely to contain sensitive information, including user and PII data. Decide in advance where content goes: for example, redacted content in the trace store under a short retention period, and full content only in a restricted store.
- **Record both the requested model and the model that answered.** The spec separates `gen_ai.request.model` from `gen_ai.response.model`. For a vendor model, the response attribute is supposed to be the exact model that actually served the request. That difference is how question 3 gets detected.

Trap to name: if you trace only the LLM calls, a failing agent looks healthy. The usual failures are tool errors, retries and loops. A trace with 40 `execute_tool` spans where you expected 4 tells you what went wrong, and a single latency number hides it.

</details>

---

### 2. Offline evals pass. Why do you still need online evaluation, and what does it measure?

`foundational`

<details>
<summary>Answer</summary>

An offline suite checks the inputs you thought to write down. Production sends inputs you didn't think of, and its distribution changes over time: new user groups, new document formats, a marketing campaign that brings in a different kind of question. Online evaluation measures quality on real traffic, where there is no reference answer.

Build it in layers, starting with the cheapest:

1. **Deterministic checks on every request.** Schema validity, whether tool calls parse, refusal and empty-output rates, citation-link validity, length. These are cheap, certain and fast to alert on.
2. **Implicit user signals.** Regenerate clicks, edits to the output, abandonment, escalation to a human, thumbs up or down. These are noisy and biased toward users who complain, so watch trends and don't treat any single value as truth.
3. **A sampled LLM judge.** Score a stratified sample, over-weighting new features, low-confidence outputs and rare segments, using a judge validated as in [evaluation.md question 1](evaluation.md#1-whats-the-validation-protocol-for-an-llm-as-judge). It runs asynchronously and stays out of the request path.
4. **Human review of a small sample** each week. This is what keeps the judge calibrated.

Senior point: **online evaluation finds problems and offline evaluation stops them from coming back.** When online eval shows a failure, it should turn into an offline test case (question 7). If a team has dashboards but its offline suite never grows, it is watching the same failures repeat.

</details>

---

### 3. Your provider may change a model behind the same ID. How would you detect a silent regression?

`single-source`

<details>
<summary>Answer</summary>

There is published evidence that this happens. Chen, Zaharia and Zou (arXiv 2307.09009, "How is ChatGPT's behavior changing over time?", preprint) compared the March 2023 and June 2023 versions of GPT-3.5 and GPT-4. GPT-4's accuracy at telling prime from composite numbers fell from **84% to 51%**, and GPT-3.5 improved on the same task over the same period. The authors trace much of the drift to a drop in how well GPT-4 followed instructions, and conclude that the "same" LLM service needs continuous monitoring.

Detection has four layers:

- **Pin dated snapshots where the provider offers them,** and treat floating aliases (`-latest`-style names) as something you accept on purpose. Log `gen_ai.response.model` next to the requested model (question 1), so that a change in which model actually served the request shows up as a data change and not as a guess.
- **Run a canary eval on a schedule, not only on deploy.** Run a fixed, versioned set of prompts against production configuration every day at low temperature, and compare the score distributions with the last known-good run. The scheduled run matters: a vendor-side change happens when nobody has deployed anything, so eval gates attached to deploys never fire.
- **Watch behavioural metrics that don't depend on a judge.** Output length, refusal rate, format-violation rate, tool-call count per task, and average token usage. A shift in any of these is a warning even while quality scores look fine, and it is often visible before them.
- **Compare against a control.** Keep a small slice of traffic, or the canary set, on a second pinned snapshot, so you can tell "the model changed" apart from "our traffic changed".

Trap: blaming the provider is the last conclusion to reach. Rule out your own prompt, retrieval-index, tool-schema and traffic-mix changes first. Your config version history (question 4) is what lets you rule them out quickly.

</details>

---

### 4. How do you version and roll out a prompt change?

`foundational`

<details>
<summary>Answer</summary>

**Version the whole configuration, not just the prompt text.** The unit that changes behaviour is the prompt template plus model ID, sampling parameters, tool definitions, retrieval settings and guardrail thresholds. Give that bundle a content hash and store it in version control or a config registry, and stamp the version on every trace. Otherwise you can't answer "which config produced this output?" during an incident.

Roll out in increasing risk:

1. **Offline gate.** The regression suite passes, including cases gathered from past incidents (question 7).
2. **Shadow.** Run the new config on a copy of live traffic, discard its outputs, and compare them with the current config using judges and deterministic checks. This carries no user risk but costs double inference on the shadowed share. It can't evaluate multi-turn flows, because the conversation follows the old config's answers, and it must not run tools with side effects.
3. **Canary.** Serve 1-5% of traffic, with automatic rollback on guardrail-rate, error-rate or latency SLO breaches (question 8). Assign traffic by user or session, not per request, so no one sees two personalities in one conversation.
4. **A/B test**, only when the question is "which is better?" and not "is anything broken?". Pick a product metric in advance and size the test for it. Quality differences between prompts are often small, and a noisy thumbs-up rate needs far more traffic than people expect.

Senior point: **rollback has to be a config flip, not a redeploy.** If reverting a prompt needs a build, the gap between detecting a problem and fixing it is measured in hours.

</details>

---

### 5. Your production model has a retirement date. Walk through the migration playbook.

`single-source`

<details>
<summary>Answer</summary>

Start with the provider's actual commitments, because they set your timeline. As of 2026-09-25:

- **Anthropic** (platform.claude.com model-deprecations page) defines four lifecycle states: Active, Legacy, Deprecated and Retired. Requests to retired models fail. It promises **at least 60 days' notice** before retiring publicly released models, and says deprecated models are "likely to be less reliable" than active ones. The dates cover Anthropic-operated platforms. Amazon Bedrock and Google Cloud set their own retirement schedules, so the same model can have a different retirement date on each platform. Example from its history: Claude Opus 4.1 developers were notified on June 5, 2026 and the model was retired on August 5, 2026. The console usage export, broken down by API key and model, is the documented way to find remaining callers.
- **OpenAI** (developers.openai.com/api/docs/deprecations) promises at least **6 months** for generally available models, at least **3 months** for specialized variants, and possibly **as little as 2 weeks** for preview models. Safety or compliance reasons can shorten any of these. Its own recent entry is a reminder that short notice happens: `gpt-5.4-cyber` was listed on 2026-09-11 with a shutdown date of October 1, 2026.

The playbook:

1. **Inventory.** Find every caller, including scripts, CI jobs, eval harnesses and the judge model itself. Callers that nobody looks at are the ones that break. Provider usage exports and your `gen_ai.request.model` traces are the source of truth; a grep of the code isn't enough.
2. **Evaluate the replacement on your eval set,** not on the vendor's benchmark. Expect to re-tune prompts: a newer model is a different model, and prompts tuned against the old model's quirks often get worse.
3. **Re-validate the judge** if it changed or if the model it grades changed. Agreement with human labels has to be measured again.
4. **Re-baseline cost and latency.** Tokenizer, verbosity and caching behaviour can all differ.
5. **Roll out with shadow and canary** (question 4), and keep the old model reachable until the canary has run long enough to show rare failures.
6. **Finish well before the shutdown date.** Aim to be done in the first half of the notice window, so an unexpected regression still leaves time to recover.

Senior point: **plan migrations as recurring work, not surprises.** A system that pins models will migrate every year or so. Teams that have automated steps 1-4 treat a deprecation notice as a routine task.

</details>

---

### 6. An LLM feature is misbehaving in production. How do you run the incident?

`foundational`

<details>
<summary>Answer</summary>

Standard incident practice still applies: an incident commander, a timeline, mitigation before root cause, and a blameless review. Three things are specific to LLM features.

**Severity depends on harm, not only on availability.** An LLM feature can be fully up while leaking PII, producing defamatory text, or calling a tool with the wrong arguments. Define in advance which output classes page someone: data exposure, unsafe content, and incorrect irreversible actions. That list decides whether a bad output is a ticket or a sev-1.

**Mitigations, fastest first:**
1. A kill switch or fallback to a deterministic path, such as "we can't help with that right now, here is a human".
2. Roll back the config version (question 4).
3. Turn off the specific tool or capability involved, rather than the whole feature.
4. Tighten a guardrail threshold, accepting more false positives for a while.
Each of these must already exist as a flag before the incident. Building a kill switch during an outage takes too long.

**Diagnosis follows the config and the trace.** Filter traces by config version and `response.model`, then check these in order: our change, then retrieval or data change, then traffic change, then provider change (question 3). Non-determinism makes reproduction hard, so save the exact inputs, retrieved context and parameters from the failing traces at the time. With content capture turned off you may have nothing to replay, which is an argument for a short-retention, access-controlled content store.

The review closes with **a new eval case for every distinct failure** (question 7) and a check on whether a monitor should have caught it earlier.

</details>

---

### 7. How do production traces become eval cases without polluting the eval set?

`foundational`

<details>
<summary>Answer</summary>

The loop: **signal → triage → label → add to the suite → gate.** Signals are online-eval failures, negative feedback, escalations and incidents. A human groups them into failure modes (the error-analysis practice in [evaluation.md question 4](evaluation.md#4-how-much-of-a-genai-project-should-be-evaluation)), labels a reference outcome or pass criterion, and adds representative cases to the regression suite that gates deploys.

The discipline that makes it work:

- **Stratify, don't dump.** Adding every thumbs-down skews the suite toward whatever users complain about most. Cap cases per failure mode and keep a fixed, random "everyday traffic" slice, so a better score reflects real improvement and not only a better fit to past complaints.
- **Keep regression sets and tuning sets separate.** If people tune prompts against the same cases that gate deploys, the gate fills up with overfitting. Hold out a portion that nobody tunes against.
- **Scrub and get consent before promotion.** Production traces contain user data. Redact PII, check that retention and consent terms allow long-term storage, and record provenance so a case can be removed if a user asks for deletion.
- **Version the eval set** together with the config (question 4), so a score change can be traced to either the system or the test.
- **Retire stale cases** whose expected answer depended on facts or policies that have since changed.

Senior point: **the eval set is a product with its own owner.** If nobody owns it, it either stops growing or grows without control, and both make the deploy gate meaningless.

</details>

---

### 8. What SLOs would you set for an LLM feature, and how do guardrails fit in?

`single-source`

<details>
<summary>Answer</summary>

Standard SLOs still apply: availability and error rate. Latency needs splitting, and quality needs SLOs of its own.

**Latency.** Set streaming and non-streaming targets separately. For streamed chat, what users experience is time to first token plus the rate of tokens after it, so set a p95 target on each. The OpenTelemetry GenAI metrics (Development status, question 1) define histograms that match this: `gen_ai.client.operation.time_to_first_chunk`, `gen_ai.client.operation.time_per_output_chunk` and `gen_ai.client.operation.duration`, plus server-side `gen_ai.server.time_to_first_token`. For agents, set the SLO on the whole task (`gen_ai.invoke_agent.duration`), because one slow tool or an extra loop dominates the total and a per-call p95 hides it.

**Guardrails have to be inside the latency budget.** An input classifier plus an output check can add as much time as the model call. Choices:
- Run the input check in parallel with generation and cancel on a flag.
- Check streamed output in chunks and stop early.
- Allow asynchronous, after-the-fact checks only for low-risk content.
Decide fail-open or fail-closed per risk class in advance. When the guardrail service times out, a support bot might fail open, but a tool that moves money fails closed.

**Guardrail and quality SLOs.** Track the guardrail block rate (a sudden change either way means something broke), the false-positive rate on a labelled sample (over-blocking is a quality failure too), and a judge-based quality pass rate on sampled traffic (question 2). A block rate that doubles overnight, with no change on your side, is a signal worth paging on.

Trap: don't set a quality SLO on a noisy judge without error bars. A 2-point drop on 200 sampled traces can be noise. Size the sample so the SLO threshold is larger than the measurement's own variance.

</details>
