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
