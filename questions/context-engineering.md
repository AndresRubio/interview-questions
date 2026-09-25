# Context engineering

**What goes into the window, turn by turn, is now the main thing you design.** [agent-architecture.md](agent-architecture.md) covers the loop and the harness, and its question 6 names the context-management mechanisms. This file goes one level down: why long contexts degrade, how compaction and memory fail, how to decide when to compact, and how output format fits in. For what caching costs and saves, see [cost-and-caching.md](cost-and-caching.md). For retrieval as a context source, see [retrieval-and-rag.md](retrieval-and-rag.md).

The senior framing to reach for: **the context window is a budget, not a bucket.** Each token you add costs attention, latency and money, and it can make the answer worse.

---

### 1. What is context engineering, and how is it different from prompt engineering?

`single-source`

<details>
<summary>Answer</summary>

Prompt engineering is about writing a good instruction. Context engineering is about **choosing, across a whole multi-turn run, the set of tokens the model sees at each inference step**. That set includes the system prompt, tool definitions, retrieved documents, tool results, message history and memory.

Anthropic's engineering post *Effective context engineering for AI agents* frames it as curating "the smallest possible set of high-signal tokens" for the outcome you want. Its toolkit:

- **System prompts** pitched at the right altitude: specific enough to steer, not hardcoded if-else logic.
- **Minimal, non-overlapping tools.** If a human can't tell which tool applies, the agent can't either.
- **Just-in-time retrieval.** Keep lightweight references such as file paths and queries, and load the content only when it's needed.
- **Long-horizon techniques:** compaction, structured note-taking, and sub-agents.

Why the distinction matters in an interview: in an agent, most tokens in the window were **not written by you**. They are tool output and history that the loop piled up. A perfect prompt is soon a small share of a 150K-token window full of stale `grep` results. The engineering work is controlling that growth.

</details>

---

### 2. Long-context models advertise 1M tokens. Why not just put everything in?

`single-source`

<details>
<summary>Answer</summary>

Because accuracy is not flat across the window, and effective capacity is well below the advertised size.

- **Lost in the middle.** Liu et al., *Lost in the Middle: How Language Models Use Long Contexts* (TACL 2023, arXiv 2307.03172), found a **U-shaped curve**: performance is highest when the relevant information sits at the start or end of the input, and drops sharply when it is in the middle. They showed this on multi-document QA and key-value retrieval, including with models built for long context.
- **Context rot.** Chroma's *Context Rot* report tested **18 models** from Anthropic, OpenAI, Google and Alibaba. It found performance becomes steadily less reliable as input grows, even on simple tasks. Three results worth knowing:
  - **Distractors compound.** One topically related non-answer hurts, several hurt more, and different distractors do different amounts of damage.
  - **Lower question-answer similarity makes length hurt faster.** Needle-in-a-haystack tests with exact lexical matches make models look better than they are.
  - **Shuffled haystacks beat coherent ones**, across all 18 models. This is counterintuitive, and it's a good sign the mechanism isn't well understood yet.
- Anthropic's post gives the intuition of an **attention budget**: every token attends to every other, so each added token spreads attention thinner.

Two independent sources agree on the direction, but how large the effect is depends heavily on the model and task. Don't quote one number as universal.

**The trade-off answer:** long windows are good for *occasional* whole-document reasoning. They are a poor default for an agent's working memory. Put the critical material at the edges, keep the middle lean, and run your own evals at *your* context lengths rather than trusting needle-in-a-haystack scores.

</details>

---

### 3. How do you compact a long agent transcript, and how does compaction fail?

`single-source`

<details>
<summary>Answer</summary>

There are three broad strategies. Each loses information differently.

**1. Summarise (rewrite).** An LLM rewrites the history into a summary. Anthropic's post describes keeping architectural decisions, unresolved bugs and implementation details while discarding redundant tool output. The failure mode is **lossy paraphrase**: exact file paths, error strings, numbers and code get rounded off. And you can't undo it.

Simon Willison gives a concrete case. A 27-minute ChatGPT agent session produced routes and map files. When he later asked for the Python code it had run, the code was gone because the thread had been compacted. His conclusion: any system that compacts should **keep the pre-compaction text and let the agent retrieve it with a tool call**.

**2. Prune (delete, keep the rest verbatim).** `fast-jev-compaction` doesn't summarise. A small model scores every tool call and result on two yes/no questions: should the call stay, and should its full result stay? Stale items are truncated or dropped. User and assistant text stays **verbatim and in order**. You lose whole items, but nothing kept gets distorted.

**3. Truncate mechanically.** AWS's Strands Harness post credits its token efficiency mainly to defaults: **tool results over ~1,500 tokens get truncated**, compaction **triggers above 85% of the window**, and an in-loop recovery step handles overflow. This is cheap and predictable. It's also blind to content.

**Opinionated take:**

- Prune before you summarise. Tool output is where the tokens are, and it's the easiest thing to re-fetch.
- Never make compaction the *only* copy. Store the raw transcript somewhere the agent can query.
- Decide ahead of time what must survive: decisions, constraints, open tasks, and exact identifiers. Test for it. "Can the agent still recite the constraint from turn 3 after two compactions?" makes a good eval.

</details>

---

### 4. When should an agent compact: on a fixed schedule, at a threshold, or when the model decides?

`single-source`

<details>
<summary>Answer</summary>

**Fixed interval or threshold** (every N tokens, or at X% full) is easy to reason about, and it's what most harnesses ship. Strands, for example, triggers at 85%. The problem is that **it's blind to the task**. It can fire in the middle of a derivation and throw away the half-built argument you needed.

**Learned or model-judged readiness** asks whether this is a good moment. *Self-Compacting Language Model Agents* (Li et al., arXiv **2606.23525**, 2026) gives the model two things. The first is a compaction tool. The second is a rubric saying when to fire (a sub-task has resolved, or the trajectory is converging) and when to hold off (mid-derivation, or stuck). For math, the rubric is checked at each round boundary, and a round is up to 16,384 tokens.

On IMO-AnswerBench with Qwen3-30B-A3B-Instruct, the scores were **52.1 for SelfCompact, 48.7 for fixed-interval and 45.2 for no compaction**. Gains were bigger on smaller models with thinking disabled. The authors say it matches or beats fixed-interval summarisation at **30-70% lower cost per question**. The paper also reports that giving the model the tool *without* the rubric works poorly, because models fire it at bad moments or never.

A further step is **training the policy in**. Tencent's ContextPilot-14B (a Qwen3 fine-tune) is trained with RL to plan, keep long-term memory, and "soft-offload" low-value context in the middle of a task.

**Trade-off:** judged readiness costs a probe per check and adds a behaviour you must evaluate. Fixed triggers cost nothing to decide and are easy to debug. A sensible default is a hard threshold as the backstop, with model-judged compaction below it for long reasoning tasks. These results come from single papers on math and search benchmarks, so check them against your own workload.

</details>

---

### 5. Design memory for an agent. Files and a database, or a vector store?

`single-source`

<details>
<summary>Answer</summary>

Split memory by **how it's read**, not by storage technology.

- **Working memory** is the context window itself. It's small and expensive, and it rots (Q2).
- **Structured state** covers things like the task list, config, decisions and user preferences. Store it in **files or a relational DB**, and read it by key or path. Anthropic's post recommends **structured note-taking**: the agent writes notes outside the window and reads them back later. Data read this way is exact and easy to inspect and diff.
- **Episodic or semantic recall** answers questions like "have I seen something like this before?". This is where **vector search** helps, because you don't know the key in advance.

Cal Paterson's **Memoryfields** makes the low-mechanism case. His argument is that memory should be a data format rather than a multi-stage pipeline. Memories are Markdown pages with YAML frontmatter, with a soft 8KB limit per page, indexed by SQLite vectors and shipped as a ZIP. Retrieval is one parallel semantic search rather than several sequential steps through a knowledge graph. He criticises graph-heavy designs as over-engineered.

The other side is the crowded graph-memory market. A Decoding AI teardown names Graphiti, mem0, cognee, HydraDB and Neo4j's agent memory. It concludes that "nobody has cracked it yet" and recommends owning the business logic through an SDK rather than building from scratch or buying a whole platform.

Memory also has to be **surfaced at the right moment**, not just stored. Meta's Proactive Memory Agent (as reported by *The Batch*, issue 371) runs next to the acting agent. It keeps notes on facts, fixes, failed commands and open tasks, and injects reminders at chosen moments. Reported gains include Claude Sonnet 4.5 on Terminal-Bench 2.0 rising from **37.6% to 45.9%**, without retraining.

**Opinionated take:** start with files plus a DB. Add vectors only when you have a real fuzzy-recall need. Treat graph memory as something you have to justify. Whatever you choose, **memory the agent never reads back is just logging**.

</details>

---

### 6. How do you keep an agent's prompts cache-friendly?

`foundational`

<details>
<summary>Answer</summary>

Prefix caching only pays off when the start of the prompt is **byte-identical** across calls. So:

- **Order by volatility.** Put the stable parts first (system prompt, tool definitions, long reference docs) and the volatile parts last (the latest turn, fresh tool results).
- **Don't put timestamps, request IDs or shuffled tool lists near the top.** One changed byte early on invalidates everything after it.
- **Append, don't rewrite.** Editing history in place, *including compaction*, breaks the cache from the edit point onward. That's a hidden cost of aggressive compaction. It's another reason to compact rarely and all at once rather than trimming a little every turn.
- **Treat config as part of the prefix.** For example, Anthropic's structured-outputs docs note that changing the output format invalidates the prompt cache for that thread.

Pricing, TTLs and hit-rate numbers are covered in [cost-and-caching.md](cost-and-caching.md).

</details>

---

### 7. Why use subagents for context, rather than just for parallelism?

`single-source`

<details>
<summary>Answer</summary>

[agent-architecture.md](agent-architecture.md) (Q1, Q6) covers subagents as a harness part. The context-engineering point is that **a subagent is a disposable context window**. It can spend 50K tokens exploring a codebase, then return a 1-2K-token summary. The orchestrator never sees the exploration, so none of that output causes rot in its window. Anthropic's post describes this pattern: specialised sub-agents with clean windows return condensed results to a coordinator.

The trade-offs a senior candidate should name:

- **The handoff is itself a compaction**, with all of Q3's failure modes. If the subagent's summary leaves something out, the orchestrator can't know it's missing.
- **Total tokens go up.** You pay for every subagent's window. You gain orchestrator quality, not efficiency.
- **Subagents don't share context.** Siblings can make conflicting assumptions. Tasks that need tight coordination fit subagents badly. Read-heavy, separable tasks such as search, review and investigation fit well.

The rule of thumb: delegate when **the work is large but the answer is small**.

</details>

---

### 8. Structured outputs: constrained decoding, validate-and-retry, or free text? And can strict formats hurt quality?

`single-source`

<details>
<summary>Answer</summary>

**Constrained decoding.** OpenAI's Structured Outputs (`strict: true`) and Anthropic's structured outputs (`output_config.format` JSON schemas, plus `strict` tool use) both limit sampling to tokens the schema's grammar allows. The output is guaranteed to parse and conform.

Costs and caveats, from the docs:

- Only a subset of JSON Schema is supported. Anthropic doesn't support recursive schemas, numeric bounds, or string length limits.
- The first request pays a grammar-compilation delay. Anthropic caches compiled grammars for 24 hours.
- The schema adds input tokens.
- Refusals are special. OpenAI returns a separate `refusal` field instead of schema-shaped output.
- OpenAI's older **JSON mode** guarantees valid JSON but **not** schema adherence. Don't mix the two up.

**Validate-and-retry.** Generate freely, then parse against a schema (Pydantic, Zod) and retry with the error message on failure. It works with any model and can check *semantic* rules that grammars can't, like "end date after start date". The price is extra latency and cost on failures, and possible retry loops.

**Can the format hurt reasoning?** Tam et al., *Let Me Speak Freely? A Study on the Impact of Format Restrictions on Performance of Large Language Models* (arXiv 2408.02442, 2024), found that **stricter format constraints generally cause larger drops on reasoning tasks**. Their comparison covered format-restricted outputs such as JSON and XML against free-form responses. Treat this as one study. How much it applies to current constrained-decoding stacks is untested here.

**The practical design** that follows: **reason first, format second.**

- Put a free-text `reasoning` field *before* the answer fields in the schema, so the model thinks before it commits.
- Or run two steps: reason in free text, then use a cheap constrained call to extract the result into the schema.
- Use constrained decoding when a parse failure is costly, such as a tool call or a downstream system. Keep validation for the semantic rules anyway.

</details>

---

**Sources** (all fetched 2026-09-25): [Anthropic, Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) · [Chroma, Context Rot](https://www.trychroma.com/research/context-rot) · [Liu et al., Lost in the Middle](https://arxiv.org/abs/2307.03172) · [Li et al., Self-Compacting Language Model Agents](https://arxiv.org/abs/2606.23525) · [Simon Willison](https://simonw.substack.com/p/navierstokes-rubygems-attacked-gis) · [fast-jev-compaction](https://github.com/tamaratran/fast-jev-compaction) · [ContextPilot-14B](https://huggingface.co/tencent/ContextPilot-14B) · [Strands Harness](https://strandsagents.com/blog/introducing-strands-harness/) · [Memoryfields](https://calpaterson.com/memoryfields.html) · [Decoding AI, unified memory](https://www.decodingai.com/p/how-to-implement-a-unified-memory-from-scratch) · [The Batch 371](https://www.deeplearning.ai/the-batch/issue-371) · [OpenAI Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs) · [Anthropic structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs) · [Tam et al., Let Me Speak Freely?](https://arxiv.org/abs/2408.02442)
