# Agent architecture

Loops, harnesses, tool design, and why agent benchmark numbers should be treated with suspicion.

---

### 1. Build me an agent loop from scratch. What are the parts?

`single-source`

<details>
<summary>Answer</summary>

Simon Willison's operational definition is the frame worth using: **an agent runs tools in a loop toward a goal**, and the skill is in designing *the tools and the loop* — not in the model call.

Parts worth naming:

- A **tool registry** with typed schemas
- A **permission layer** deciding what runs unattended versus what needs approval
- **Live context assembly** — what goes into each turn
- **Context reduction / compaction** as the window fills
- **Transcript persistence** so runs resume rather than restart
- **Bounded subagents** for parallelism and context isolation
- A **termination condition** that is more than a turn cap

Public reference implementations to point at:

- `rasbt/mini-coding-agent` — single-file, standard-library-only Python harness with live repo context, permissioned tools, context reduction, transcript resumption and bounded subagents
- `huggingface/smolagents` — roughly 1,000 lines of core code including sandbox isolation via E2B, Docker or Pyodide

</details>

---

### 2. Where do agentic loops actually pay off?

`single-source`

<details>
<summary>Answer</summary>

Where **success criteria are checkable and iteration is cheap.**

The canonical wins: debugging against failing tests, performance benchmarking, dependency upgrades, slimming container images. The common amplifier across all of them is **automated tests** — the loop needs a signal it can grind against without a human in it.

The corollary is the more useful half of the answer: tasks with **no cheap verifier** are where agent loops burn tokens and produce confident garbage. Being able to say which side of that line a task falls on is the judgement actually being tested.

</details>

---

### 3. Model or harness — which moves agent performance more?

`single-source`

<details>
<summary>Answer</summary>

Published 2026 evidence points at the **harness**.

- LangChain moved a coding agent from **rank 30 to top 5 on Terminal Bench 2.0 with no model swap**
- Their July 2026 Nemotron 3 Ultra playbook reports harness-only tuning reaching **within one point of Opus 4.8** on Deep Agents at roughly **one-tenth the cost — $4.48 vs $43.48**

The implication for interviews: talking about which model you'd choose is a weak answer. Talking about tool design, context budget, retry structure and verification is the strong one.

</details>

---

### 4. Why are agent benchmark results untrustworthy, and what would you do about it?

`single-source`

<details>
<summary>Answer</summary>

Three documented failure modes, all quantified:

- **Lucky passes.** AgentLens found up to **23.2%** of SWE-agent passes were lucky across **2,614 OpenHands trajectories** — enough to shift model rankings by up to **five positions**.
- **Environment sensitivity.** Anthropic measured **6+ percentage-point** benchmark swings from container resource configuration alone.
- **Contamination.** Claude Opus 4.6 was documented identifying a benchmark by name and decrypting its answer key during BrowseComp.

What to do:

- Run **network-isolated** — this is a stated harness requirement, not a nicety
- **Pin and report container resources**
- **Inspect trajectories** rather than trusting pass/fail
- Report **variance across seeds**, not a single number

</details>

---

### 5. What's your credential hygiene for an agent that touches real systems?

`single-source`

<details>
<summary>Answer</summary>

Bound the blast radius by construction, not by hoping the agent behaves.

Willison's prescription: give agents **staging or test credentials**, and **cap spend** anywhere money is involved. His concrete illustration is a dedicated Fly.io organisation with a **$5 budget limit** and a scoped API key.

Generalised: separate identity per agent, least privilege on every token, hard spend caps, and human approval on anything with egress or irreversible effect.

This connects directly to the MCP security material — the exfiltration research obtained its results with human approval **deliberately disabled**, which tells you how much that control is carrying.

</details>

---

### 6. How do you manage context in a long-running agent?

`single-source`

<details>
<summary>Answer</summary>

Two mechanisms worth naming:

- **Compaction** — summarising earlier messages as the window fills. Worth knowing this is vendor-shipped in the Claude Agent SDK rather than something you must invent.
- **Subagents** — which buy parallelism and, more importantly, **context isolation**: a subagent burns its own window and returns only a result to the orchestrator.

The design question underneath, and the one that shows judgement: **what must survive compaction?** Decisions and constraints, yes. Raw tool output, usually not. Answering that rather than just naming the mechanisms is what the question is for.

</details>

---

### 7. Build an agent that uses tools reliably. Where do you start?

`single-source`

<details>
<summary>Answer</summary>

Start with the tools, because most tool-call failures I've seen trace back to tool design rather than the model. Anthropic's engineering post "Writing effective tools for agents" makes the same argument: a tool is a contract written for a non-deterministic caller, so you design it the way you'd write docs for a new colleague.

**Tool design.** Fewer tools with clear names beat a thin wrapper over every API endpoint. If the agent always calls `list_users` and then `get_user`, give it one `find_user(query)` tool that does the search and returns the relevant fields. Namespacing (`crm_search_contacts`, `billing_get_invoice`) helps once you have many tools and the model starts confusing similar ones. The trade-off is that consolidated tools are less general, so you'll write more of them per workflow. I accept that, because every extra tool in the list is one more thing the model can pick wrongly.

Descriptions are prompts. Say what the tool does, when to use it instead of its neighbour, what each argument means with units and formats, and what comes back. Tight schemas do part of that work for free: enums instead of free strings, required fields marked as required, `user_id` rather than `user`. Return only what the model needs, with a `response_format` or page-size option if the output can be large.

Error messages should tell the model what to do next. `400 Bad Request` gives it nothing. `"date must be YYYY-MM-DD; got '3rd March'"` or `"No customer matches 'Acme'. Try search_customers with a partial name."` lets it fix the call on the next turn.

**Validate before executing.** Check arguments in the harness against the schema and against business rules (does this account exist, is the amount under the limit, is this path inside the workspace) before anything runs. A rejected call that returns a clear error costs one turn. A malformed call that half-executes costs a cleanup.

**Idempotency and retries.** Models repeat calls, and harnesses retry on timeouts, so assume every tool can be called twice. Read tools are naturally safe. For writes, accept an idempotency key (derived from the run ID and step) so a retried `create_ticket` returns the existing ticket instead of opening a second one. Retry transient failures (timeouts, 429s, 5xx) in the harness with backoff and don't show them to the model. Return validation and permission errors to the model, since only it can change the arguments.

**Strict tool calling.** Both Anthropic and OpenAI offer a strict mode (`strict: true` on the tool definition) that constrains decoding so arguments always match the JSON schema. Turn it on where it's supported. It removes a whole class of parse failures, but it only guarantees shape. The model can still pick the wrong tool or pass a valid-looking wrong ID, so it doesn't replace validation. Strict mode also supports only a subset of JSON Schema, which can push you toward simpler schemas (that's usually fine, see [context-engineering.md question 8](context-engineering.md#8-structured-outputs-constrained-decoding-validate-and-retry-or-free-text-and-can-strict-formats-hurt-quality) on whether strict formats hurt quality).

**Measure tool-call accuracy.** Build an eval set of realistic tasks from production traces and score several things separately: did it pick the right tool, were the arguments right, did it call tools it didn't need, how many calls did it take, and did the task succeed end to end. The Anthropic post recommends exactly this loop: run the eval, read the transcripts, and let the failures tell you which descriptions to rewrite. Public benchmarks are useful reference points. BFCL (the Berkeley Function Calling Leaderboard) checks arguments by AST match and by execution, has a relevance-detection category where the right answer is no call at all, and since v3 includes multi-turn tasks. tau-bench runs multi-turn conversations with a simulated user and a policy, and its pass^k metric (success on all k repeated trials) shows how much reliability falls off when you ask for consistency rather than one lucky pass. For your own agent, report pass^k too.

Sources: Anthropic, "Writing effective tools for agents" (anthropic.com/engineering/writing-tools-for-agents); Anthropic strict tool use and OpenAI function-calling docs; BFCL; tau-bench. All fetched 2026-11-02.

</details>

---

### 8. Your agent gets stuck in a loop. How do you fix it?

`foundational`

<details>
<summary>Answer</summary>

First find out what kind of loop it is, because the fix depends on the cause. I'd pull the trace (see [llmops.md question 1](llmops.md#1-how-would-you-trace-an-agent-run-end-to-end-and-what-is-the-actual-status-of-the-opentelemetry-genai-conventions) on tracing agent runs) and look at the last twenty steps before changing anything.

**Detection.** Three signals cover most cases, and the harness should check all of them on every step:

- Repeated identical calls: hash the tool name plus normalised arguments, and flag the same hash three times in a window. Also catch near-repeats, such as the same search with one word changed.
- No progress in state: track something that should move (files changed, tests passing, items processed, new information in tool results). If it hasn't changed in N steps, the agent is spinning even when the calls look different.
- Budgets: a step cap, a token or dollar cap, and a wall-clock cap per run. These are the backstop for loops the other two checks miss.

**Causes.** The common ones are in the harness, not the model.

- Unhelpful tool errors. If a tool returns `error` or an empty list, the model often retries the same call because nothing told it what to change. This is the most frequent cause I'd expect, and it's fixed in the tool (question 7).
- Ambiguous success signals. The agent can't tell it's done, so it keeps verifying. A write tool that returns nothing, or tests that pass but print warnings, invite another check. Make tools return explicit confirmation and give the agent a clear done condition.
- Context loss after compaction. The summary dropped the fact that an approach already failed, so the agent tries it again. See [context-engineering.md questions 3](context-engineering.md#3-how-do-you-compact-a-long-agent-transcript-and-how-does-compaction-fail) and [4](context-engineering.md#4-when-should-an-agent-compact-on-a-fixed-schedule-at-a-threshold-or-when-the-model-decides): keep a short log of attempted approaches and their outcomes in the part of the state that survives compaction.
- Model-level repetition. Reasoning models can also loop inside their own thinking. That's a model fix, and you usually can't apply it to a hosted model, so the harness still needs its own checks.

**Recovery.** Escalate in steps, since each one costs more than the last.

1. Inject a hint. When the detector fires, add a message saying what it saw: "You've called `search_docs` with the same query three times and got no results. Try a different query or a different tool, or say what's blocking you." This often works because it gives the model the progress signal it was missing.
2. Escalate to a human if the hint doesn't help, or immediately for tasks where a wrong guess is expensive. Hand over the goal, what was tried, and where it's stuck.
3. Hard-stop with a partial result when budgets run out. Return what got done, what didn't, and why, instead of a bare failure, so the work isn't lost.

The trade-off is between autonomy and cost. Aggressive thresholds interrupt runs that were about to succeed, while loose ones burn money. I'd set them from real traces: look at how many steps successful runs take, and put the cap somewhat above the long tail.

**Prevention through harness design.** Make loops less likely and cheaper when they happen. Write tools whose errors say what to change. Make tools idempotent so a repeat is harmless. Keep an explicit task state (a plan or checklist the agent updates) so progress is visible to both the agent and the detector. Persist that state through compaction. Give subagents their own budgets so one stuck branch can't drain the run. Then turn every loop you catch in production into an eval case, so the fix stays fixed.

</details>
