# Senior and behavioural

The rounds candidates under-prepare and most often fail.

---

### 1. How do you use coding agents, and where do they fail?

<details>
<summary>Answer</summary>

Answer with a **measured claim and a limit**, not enthusiasm.

Where they pay off: tasks with **cheap, checkable success criteria** — failing tests, dependency upgrades, benchmark-driven optimisation.

Where they fail: anything without a verifier, large ambiguous refactors, and work where being confidently wrong is expensive.

Then the review discipline, which is what's actually being probed: you read the diff, you don't merge what you can't explain, and you treat agent output as a draft from a fast junior. Roughly **51% of committed code was AI-generated or AI-assisted as of early 2026**, which is precisely why raw commit volume has stopped being evidence of anything — and recruiters are now explicitly told to judge system-level thinking and documentation instead.

**Avoid unfalsifiable productivity multipliers.** This:

> It cut dependency-upgrade PRs from a day to an hour, and I stopped using it for schema migrations after it silently dropped a constraint.

is worth far more than "it makes me 3x faster".

</details>

---

### 2. Walk me through an architecture decision you'd defend.

`single-source`

<details>
<summary>Answer</summary>

This is the round that differentiates senior candidates. The reported 2026 differentiator is walking through decisions and trade-offs on a **named project** — not breadth of tool familiarity.

Structure:

1. The **constraint** that forced a choice
2. The options you **rejected**, and why
3. What you'd **measure** to know you were wrong
4. What you'd **do differently** now

Point 4 is the one people skip and the one that reads as senior.

Have one ready that includes a *rejection*. Choosing not to use the impressive technology, with the arithmetic to justify it, is a stronger signal than having used it — and it inoculates you against the over-engineering critique.

</details>

---

### 3. The values round. What's actually being assessed?

`single-source`

<details>
<summary>Answer</summary>

**Not your opinions about AI.**

At AI labs it's a full **one-hour scored session with non-technical interviewers**, probing personal experience and emotional reflection rather than views on the field. interviewing.io reports this is **where most candidates fail** — precisely because they prepare for it as a culture-fit chat.

Prepare it like a technical round: concrete situations, what you actually did, what you got wrong, what you changed as a result. Reflection with specifics beats values vocabulary.

</details>

---

### 4. What's the honest weakness in the system you just described?

<details>
<summary>Answer</summary>

**Name it before they find it.** Volunteering the weakness is the strongest available move — it demonstrates you understand the difference between a demo and a system, and it moves the conversation from evaluation to collaboration.

The weaknesses worth having ready for any retrieval or agent system:

- **Coverage.** Is the corpus actually complete, or is it what happened to be published? "Absence of a record is not evidence of absence" is a real limitation in most real datasets, and few candidates state it.
- **Synthetic eval bias.** Questions generated *from* passages share vocabulary with the target and are systematically easier than real ones. Such sets measure **regressions** well and **absolute quality** badly.
- **Entity resolution.** Where the natural key is broken or reused, some fraction of records simply cannot be resolved from available data — quantify that fraction rather than hiding it.
- **False-positive rate.** Any classifier or flag has one. Quote it.

The structure that works: state the limitation, say how you measured it, say what you'd need to fix it. Stating a limitation unprompted is the same instinct interviewers are testing when they ask how you'd evaluate anything.

</details>
