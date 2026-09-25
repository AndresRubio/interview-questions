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
| [OWASP Top 10 for LLM Applications 2025](https://genai.owasp.org/llm-top-10/) | LLM01–LLM10 |
| [Anthropic prompt caching docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) | Cache pricing, TTLs, minimums. Fetched 2026-09-25. |
| [OpenAI prompt caching guide](https://developers.openai.com/api/docs/guides/prompt-caching) | Cache discount, minimum tokens. Fetched 2026-09-25. |
| [OpenAI structured outputs](https://developers.openai.com/api/docs/guides/structured-outputs) | Constrained decoding, strict mode |
| [Anthropic structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs) | Grammar compilation, schema limits |
| [OpenTelemetry GenAI semantic conventions](https://github.com/open-telemetry/semantic-conventions-genai) | Agent/tool span names, metrics. Moved out of opentelemetry.io; status Development. Fetched 2026-09-25. |
| [Anthropic model deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations) and [OpenAI deprecations](https://developers.openai.com/api/docs/deprecations) | Notice periods, retirement dates. Fetched 2026-09-25. |
| Vision docs: [Anthropic](https://platform.claude.com/docs/en/build-with-claude/vision), [OpenAI](https://developers.openai.com/api/docs/guides/images-vision), Gemini [tokens](https://ai.google.dev/gemini-api/docs/tokens), [video](https://ai.google.dev/gemini-api/docs/video-understanding), [media resolution](https://ai.google.dev/gemini-api/docs/media-resolution) | Image/video/audio token accounting. Fetched 2026-09-25. |
| [OpenAI Terms of Use](https://openai.com/policies/terms-of-use) | Restriction on using Output to develop competing models. Accessed 2026-09-25. |
| [Linux Foundation press](https://www.linuxfoundation.org/press) | A2A governance, membership (promotional on adoption) |

## Peer-reviewed and preprints

| Source | Status |
|---|---|
| [Parasites in the Toolchain](https://arxiv.org/pdf/2509.06572) — MCP ecosystem attack surface | **Peer-reviewed**, IEEE S&P 2026 pp. 138–155 |
| [BM25 Wins at Scale](https://arxiv.org/abs/2607.26497) — retrieval scaling study | Preprint, Jul 2026 |
| [GraphRAG-Bench](https://arxiv.org/abs/2506.02404) | Preprint |
| [RAGSearch](https://arxiv.org/abs/2604.09666) — agentic retrieval vs GraphRAG | Preprint, Apr 2026 |
| [LLM-as-judge measurement critique](https://arxiv.org/abs/2508.18076) | Position paper |
| [Semantic Uncertainty](https://arxiv.org/abs/2302.09664) — Kuhn, Gal & Farquhar | ICLR 2023 |
| [Detecting hallucinations using semantic entropy](https://www.nature.com/articles/s41586-024-07421-0) — Farquhar, Kossen, Kuhn & Gal | Nature 2024 |
| [LoRA](https://arxiv.org/abs/2106.09685) — Hu et al. | ICLR 2022 |
| [QLoRA](https://arxiv.org/abs/2305.14314) — Dettmers et al. | NeurIPS 2023 |
| [DPO](https://arxiv.org/abs/2305.18290) — Rafailov et al. | NeurIPS 2023 |
| [InstructGPT](https://arxiv.org/abs/2203.02155) — Ouyang et al. | NeurIPS 2022 |
| [LoRA Learns Less and Forgets Less](https://arxiv.org/abs/2405.09673) — Biderman et al. | TMLR 2024 |
| [Co-Evolving Harnesses and Models](https://arxiv.org/abs/2609.09134) — Salesforce | Preprint, Sep 2026 |
| [Don't Break the Cache](https://arxiv.org/abs/2601.06007) — prompt caching for agents | Preprint, Jan 2026 |
| [GPT Semantic Cache](https://arxiv.org/abs/2411.05276) | Preprint |
| [Not what you've signed up for](https://arxiv.org/abs/2302.12173) — Greshake et al., indirect injection | AISec 2023 |
| [CaMeL: Defeating Prompt Injections by Design](https://arxiv.org/abs/2503.18813) | Preprint |
| [Design Patterns for Securing LLM Agents](https://arxiv.org/abs/2506.08837) | Preprint |
| [Lost in the Middle](https://arxiv.org/abs/2307.03172) — Liu et al. | TACL 2024 |
| [SelfCompact](https://arxiv.org/abs/2606.23525) — Li et al. | Preprint, Jun 2026 |
| [Let Me Speak Freely?](https://arxiv.org/abs/2408.02442) — Tam et al., format restrictions | EMNLP 2024 Industry |
| [How is ChatGPT's behavior changing over time?](https://arxiv.org/abs/2307.09009) — Chen, Zaharia & Zou | Harvard Data Science Review 2024 |
| [CLIP](https://arxiv.org/abs/2103.00020) — Radford et al. | ICML 2021 |
| [LLaVA](https://arxiv.org/abs/2304.08485) and [LLaVA-1.5](https://arxiv.org/abs/2310.03744) | NeurIPS 2023; CVPR 2024 |
| [POPE](https://arxiv.org/abs/2305.10355) — object hallucination | EMNLP 2023 |
| [ColPali](https://arxiv.org/abs/2407.01449) | ICLR 2025 |
| [Vision language models are blind](https://arxiv.org/abs/2407.06581) | ACCV 2024 |
| [LIMA](https://arxiv.org/abs/2305.11206) | NeurIPS 2023 |
| [AlpaGasus](https://arxiv.org/abs/2307.08701) | ICLR 2024 |
| [Deduplicating Training Data Makes Language Models Better](https://arxiv.org/abs/2107.06499) — Lee et al. | ACL 2022 |
| [FineWeb](https://arxiv.org/abs/2406.17557) | NeurIPS 2024 Datasets & Benchmarks |
| [AI models collapse when trained on recursively generated data](https://www.nature.com/articles/s41586-024-07566-y) — Shumailov et al. | Nature 2024 |
| [Is Model Collapse Inevitable?](https://arxiv.org/abs/2404.01413) — Gerstgrasser et al. | COLM 2024 |
| [UltraFeedback](https://arxiv.org/abs/2310.01377) | ICML 2024 |
| [Secrets of RLHF Part II](https://arxiv.org/abs/2401.06080) | Preprint |
| [DoReMi](https://arxiv.org/abs/2305.10429) | NeurIPS 2023 |
| [RegMix](https://arxiv.org/abs/2407.01492) | ICLR 2025 |
| [GPT-3](https://arxiv.org/abs/2005.14165) — Brown et al. | NeurIPS 2020 |

## Practitioner and civic-tech

| Source | Used for |
|---|---|
| [hamel.dev evals FAQ](https://hamel.dev/blog/posts/evals-faq/) | Judge validation, error analysis budget |
| [Simon Willison — Designing Agentic Loops](https://simonw.substack.com/p/designing-agentic-loops) | Agent definition, credential hygiene |
| [cvs-health/uqlm](https://github.com/cvs-health/uqlm) | UQ scorer taxonomy, black-box vs white-box split. Apache 2.0. Fetched 2026-09-25; repo states no benchmark results. |
| [interviewing.io — Anthropic questions](https://interviewing.io/anthropic-interview-questions) | Loop structure, values round |
| [Uncharted Career — AI interview policies](https://unchartedcareer.com/research/ai-interview-policies) | Per-company AI-use policy tracker, 6 Aug 2026. Vendor aggregation, not independent research. |
| [Anthropic — Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | Context engineering definition, compaction, subagents |
| [Chroma — Context Rot](https://www.trychroma.com/research/context-rot) | Long-context degradation across 18 models |
| [Simon Willison — The lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) and [markdown exfiltration](https://simonwillison.net/tags/markdown-exfiltration/) | Injection threat model, exfiltration channel |
| [Simon Willison newsletter, Sep 2026](https://simonw.substack.com/p/navierstokes-rubygems-attacked-gis) | Compaction losing code; OpenRouter deployment variance |
| [Spotify — Portal](https://engineering.atspotify.com/2026/9/portal-by-spotify-cut-my-claude-code-token-usage-by-90) | Routing bulk reads to a cheap model (first-party claim) |
| [Devin Fusion (Cognition)](https://cognition.com/blog/local-fusion) | Lead/executor routing (vendor claim) |
| [Google — agents challenge patterns](https://developers.googleblog.com/en/4-engineering-patterns-behind-the-strongest-ai-agents-challenge-submissions/) | Tiered routing, zero-token regex tier |
| [A cache hit is not proof](https://siddhantkhare.com/writing/kv-cache-truth-auditor) | Verifying cache-hit claims |
| [AWS Strands Harness](https://strandsagents.com/blog/introducing-strands-harness/) | Truncation and compaction trigger (vendor) |
| [fast-jev-compaction](https://github.com/tamaratran/fast-jev-compaction), [ContextPilot-14B](https://huggingface.co/tencent/ContextPilot-14B), [Memoryfields](https://calpaterson.com/memoryfields.html), [Decoding AI memory post](https://www.decodingai.com/p/how-to-implement-a-unified-memory-from-scratch) | Compaction and memory designs |
| [LangSmith Connections](https://www.langchain.com/blog/connections-managed-credentials-and-per-caller-identity-for-managed-deep-agents), [webagent](https://github.com/TheAgent-net/webagent), [Vercel Run](https://vercel.com/blog/introducing-run), [geiger](https://github.com/Atomburstofficial/geiger), [Cloudflare security-audit-skill](https://github.com/cloudflare/security-audit-skill) | Credential scoping, action chokepoints, sandboxing, fresh verifiers |
| [The Batch, issue 371](https://www.deeplearning.ai/the-batch/issue-371) | Meta memory agent and Sentinel (secondary report) |
| Prep guides: [topgenaijobs](https://www.topgenaijobs.com/blog/fine-tuning-interview-questions), [KalyanKS hub](https://github.com/KalyanKS-NLP/LLM-Interview-Questions-and-Answers-Hub), [amirteymoori](https://amirteymoori.com/ai-llm-engineer-interview-questions-2025/), [lockedinai](https://www.lockedinai.com/blog/ai-engineer-interview-questions), [rungcode](https://rungcode.io/guides/llm-interview-questions), [cyberinterviewprep](https://cyberinterviewprep.com/resources/llm-prompt-injection-defense-interview-questions), [myengineeringpath](https://myengineeringpath.dev/genai-engineer/llm-caching/) | Which questions are commonly asked. None ties a question to a named company. |
| [LiteParse](https://github.com/run-llama/liteparse) | Local PDF parsing with selective OCR |
| [POPE repo](https://github.com/RUCAIBox/POPE) | Negative-sampling settings |
| [Voice agents: a deep dive](https://boringbot.substack.com/p/voice-agents-a-deep-dive) — Hamza Farooq, Jul 2026 | 700–800 ms turn budget (only the free part of the paywalled post was checked) |

## Known weaknesses

- The files added 2026-09-25 (post-training, context engineering, cost and caching, LLM security, multimodal, LLMOps, data curation) draw their *question lists* from prep guides, not first-hand interview reports. Several 2026 examples came via the [ai-digest](https://andresrubio.github.io/ai-digest/) knowledge base and were kept only where the primary source was fetched and confirmed.
- Verification concentrated on **agent protocols**. The retrieval, evaluation and interview-process material is well-sourced but **not cross-checked** — treat `single-source` badges literally.
- Several sources are **vendor-promotional** on adoption breadth — Zed's ACP editor list, the LF A2A press release, the agentskills.io client showcase. Governance and specification facts from these are authoritative; adoption framing is not.
- Anything **version-pinned** in MCP was captured ~6 weeks after a breaking revision, while ecosystem migration was in flux.
