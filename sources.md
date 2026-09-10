# Sources

Compiled 10 September 2026. Claims were extracted from primary sources; a subset was put through three-vote adversarial verification.

Quality grading reflects the source type, not agreement with it.

## Specifications and standards bodies

| Source | Used for |
|---|---|
| [MCP 2026-07-28 changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog) | Breaking changes, deprecation lifecycle |
| [MCP security best practices](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices) | Token passthrough, confused deputy |
| [MCP Streamable HTTP transport](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http) | Statelessness, SSE resumability removal |
| [MCP MRTR pattern](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr) | Multi Round-Trip Requests |
| [agentskills.io](https://agentskills.io/home) | `SKILL.md` frontmatter spec, client showcase |
| [zed.dev/acp](https://zed.dev/acp) | ACP scope and adoption (vendor-promotional) |
| [Agent Client Protocol governance](https://agentclientprotocol.com/community/governance) | Interim BDFL model |
| [Linux Foundation press](https://www.linuxfoundation.org/press) | A2A governance, membership (promotional on adoption) |

## Peer-reviewed and preprints

| Source | Status |
|---|---|
| [Parasites in the Toolchain](https://arxiv.org/pdf/2509.06572) — MCP ecosystem attack surface | **Peer-reviewed**, IEEE S&P 2026 pp. 138–155 |
| [BM25 Wins at Scale](https://arxiv.org/abs/2607.26497) — retrieval scaling study | Preprint, Jul 2026 |
| [GraphRAG-Bench](https://arxiv.org/abs/2506.02404) | Preprint |
| [RAGSearch](https://arxiv.org/abs/2604.09666) — agentic retrieval vs GraphRAG | Preprint, Apr 2026 |
| [LLM-as-judge measurement critique](https://arxiv.org/abs/2508.18076) | Position paper |

## Practitioner and civic-tech

| Source | Used for |
|---|---|
| [hamel.dev evals FAQ](https://hamel.dev/blog/posts/evals-faq/) | Judge validation, error analysis budget |
| [Simon Willison — Designing Agentic Loops](https://simonw.substack.com/p/designing-agentic-loops) | Agent definition, credential hygiene |
| [interviewing.io — Anthropic questions](https://interviewing.io/anthropic-interview-questions) | Loop structure, values round |
| [Uncharted Career — AI interview policies](https://unchartedcareer.com/research/ai-interview-policies) | Per-company AI-use policy tracker, 6 Aug 2026. Vendor aggregation, not independent research. |

## Known weaknesses

- Verification concentrated on **agent protocols**. The retrieval, evaluation and interview-process material is well-sourced but **not cross-checked** — treat `single-source` badges literally.
- Several sources are **vendor-promotional** on adoption breadth — Zed's ACP editor list, the LF A2A press release, the agentskills.io client showcase. Governance and specification facts from these are authoritative; adoption framing is not.
- Anything **version-pinned** in MCP was captured ~6 weeks after a breaking revision, while ecosystem migration was in flux.
