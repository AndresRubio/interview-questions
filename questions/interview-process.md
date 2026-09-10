# Interview process

How the loop is actually shaped, and the one procedural question with no industry answer.

---

### 1. What does a senior AI engineering loop look like end to end?

`single-source`

<details>
<summary>Answer</summary>

At a frontier lab the shape is **conventional big-tech, not AI-specific**:

1. **Recruiter call** — ~30 minutes
2. **Coding challenge** — 60–90 minutes, most often a **90-minute asynchronous CodeSignal take-home**, sometimes skipped for referrals
3. **Onsite** — 4–5 hours across five roughly one-hour sessions: hiring manager, coding, **system design**, a second role-specific coding round, and a **values round**

End to end: roughly **3–4 weeks**.

Two things in that shape are worth preparing for specifically:

- **Classic fundamentals still gate you.** Concurrency and multithreading recur across multiple rounds, alongside hash maps, parsing, arrays, strings and sorting. GenAI knowledge does not substitute for this.
- The take-home is typically **progressive levels** where code must pass all tests at one level to unlock the next. Candidates commonly run out of time — it tests spec-reading under ambiguity and speed, not AI knowledge.

The system design round *is* AI-flavoured: LLM serving infrastructure, with prompts of the form "design a Claude chat service". See [inference-and-systems.md](inference-and-systems.md).

</details>

---

### 2. Can candidates use AI tools during interviews?

`single-source`

<details>
<summary>Answer</summary>

**There is no industry norm.** You must check per employer.

As of a policy tracker updated **6 August 2026**, across 20 tracked companies: **10 ban** AI use, **4 allow**, **4 require**, **1 discourages**, **1 has no published policy**.

| Stance | Examples | Notes |
|---|---|---|
| **Requires** | Coinbase, Canva, McKinsey, Zapier | Coinbase grades how you *direct and critique* the AI (13 Jul 2026). Canva names Copilot, Cursor and Claude for backend, ML and frontend rounds. |
| **Permits** | Meta, Shopify | Meta's rationale: closer to the real working environment. |
| **Bans in live rounds** | Anthropic | Publishes candidate-facing guidance on which stages permit AI. An AI lab banning AI in live interviews surprises people. |
| **Format as control** | Google | No published policy; reintroduced in-person rounds to verify fundamentals. |

> The tracker is a vendor-published aggregation of primary company sources, not independent survey research, and several entries are undated snapshots. Policies in this area have moved fast — re-verify before relying on any single row.

</details>

---

### 3. Are interviews moving back in person?

`single-source`

<details>
<summary>Answer</summary>

Partly, and the driver is stated openly: **anti-cheating**.

In-person rounds rose from **24% of loops in 2022 to 38% in 2025**, and frontier labs are reported to require in-person onsites. Google's approach — reintroducing at least one in-person round to verify fundamentals, without publishing an AI policy — is format used as a control instead of a rule.

Practical consequence for remote candidates: budget for travel, and raise the question early rather than discovering it at offer stage.

</details>
