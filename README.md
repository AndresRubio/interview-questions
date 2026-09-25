# AI Engineering Interview Notes

Working notes for senior AI engineering interviews, compiled September 2026.

These are not scraped listicles. Every claim was pulled from a primary source — official specifications, standards bodies, peer-reviewed papers, first-party API documentation — and a subset was put through adversarial verification, where independent reviewers actively tried to refute each claim. Claims that failed are kept in [`reference/refuted.md`](reference/refuted.md) rather than deleted, because knowing what *doesn't* survive scrutiny is worth as much as knowing what does.

The field moves fast enough that confidently repeating a stale fact is a real interview risk. Everything version-sensitive here carries a date.

## How to use this

Each question hides its answer behind a collapsible block. Read the question, answer it out loud, *then* expand. Reading answers you haven't attempted feels productive and teaches almost nothing.

```
questions/    — 42 questions with answers, grouped by topic
reference/    — cheat sheets for the things that changed recently
sources.md    — every source, with its quality grading
```

## Confidence legend

Not everything here is equally certain, and the notes say so.

| Badge | Meaning |
|---|---|
| `verified 3-0` | Survived three independent attempts at refutation. Quote freely. |
| `verified 2-1` | One reviewer dissented. Attribute it — "one controlled study found" — rather than asserting it. |
| `single-source` | Taken from a primary source but not cross-checked. Say where it came from. |
| `refuted` | Failed verification. Listed in `reference/refuted.md` so it doesn't get repeated. |
| `foundational` | Established, stable material that predates this research — textbook ML rather than 2026 ecosystem facts. Did not go through the verification pipeline because it did not need to. |

## Contents

| Topic | Questions | Notes |
|---|---|---|
| [Agent protocols](questions/agent-protocols.md) | 9 | MCP, A2A, ACP, Agent Skills. Mostly `verified 3-0`. |
| [Agent architecture](questions/agent-architecture.md) | 6 | Loops, harnesses, why benchmarks lie. |
| [Retrieval and RAG](questions/retrieval-and-rag.md) | 5 | Where the received wisdom is wrong. |
| [Evaluation](questions/evaluation.md) | 4 | Reported as the largest skill gap in the candidate pool. |
| [Inference and systems](questions/inference-and-systems.md) | 4 | Serving, cost arithmetic, observability. |
| [Senior and behavioural](questions/senior-and-behavioural.md) | 4 | The rounds where most candidates actually fail. |
| [Model evaluation](questions/model-evaluation.md) | 7 | Perplexity, bits per byte, quantization damage, contamination. |
| [Interview process](questions/interview-process.md) | 3 | Loop shape, and whether you may use AI in it. |

## Reference

- [**MCP 2026-07-28**](reference/mcp-2026-07-28.md) — the breaking revision, and what it broke
- [**Protocol landscape**](reference/protocol-landscape.md) — how MCP, A2A, ACP and Agent Skills relate
- [**Evaluation playbook**](reference/evaluation-playbook.md) — judge validation, known biases, what to skip
- [**Refuted claims**](reference/refuted.md) — plausible-sounding things that failed verification
- [**Sources**](sources.md)

## A caveat on dates

Compiled 10 September 2026. The protocol material is the most perishable: MCP's stateless revision was roughly six weeks old at the time of writing and SDK migration was still in flux. Re-check anything version-pinned before relying on it in an interview.

## Licence

Content is [CC BY 4.0](LICENSE). Use it, adapt it, credit it.
