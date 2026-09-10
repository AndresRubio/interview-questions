# Protocol landscape

How MCP, A2A, Agent Client Protocol and Agent Skills relate, as of September 2026. All `verified 3-0`.

## The four layers

| Protocol | Layer | Governance | Maturity |
|---|---|---|---|
| **MCP** | Agent → tools and data | Anthropic → Agentic AI Foundation (Linux Foundation), **2025-12-09** | Production. Breaking revision `2026-07-28`. |
| **A2A** | Agent ↔ agent, across org boundaries | Google → Linux Foundation **Jun 2025**; joined AAIF **2026-08-17** | Most institutionally mature. **v1.0 on 2026-03-12**, v1.0.1 **2026-05-28**. 150+ supporting orgs. |
| **ACP** (Agent Client Protocol) | Editor ↔ agent | Zed, Apache-2.0, no CLA. **Interim** co-governance with JetBrains, two BDFLs. Foundation move intended, **undated**. | Real but narrower. v0.13.6 (5 Jun 2026), ~3.4K stars. SDKs: Rust, TypeScript, Python, Kotlin, Java. |
| **Agent Skills** (`SKILL.md`) | Portable capability packaging | agentskills.io; reference validator `skills-ref` | **46 implementing clients** including OpenAI, GitHub, Google, Cursor, Mistral. |

## MCP and A2A are complementary, not competing

The Linux Foundation's own framing:

> A2A defines how agents communicate and coordinate with each other across organizational boundaries, while MCP defines how agents connect to internal tools and data sources.

This is now **structural** rather than diplomatic — both live under the Agentic AI Foundation, an LF directed fund co-founded with Block and OpenAI, alongside goose, AGENTS.md and agentgateway.

Practitioner disagreement exists but is about emphasis ("we already have MCP, why do we need A2A?"), not about the vertical/horizontal split itself.

## A2A v1.0 — what shipped

Released **2026-03-12**. Four named changes:

- **Multi-protocol support** via per-`AgentInterface` `protocolBinding`
- **Enterprise multi-tenancy** — `tenant` field on all request messages
- **Modernised security** — deprecated OAuth flows removed, `pkce_required` added, `DeviceCodeOAuthFlow` per RFC 8628
- **Documented three-phase migration**: Compatibility Layer → Dual Support → v1.0 Only, with AgentCards advertising v0.3 and v1.0 simultaneously

> **Caveat on adoption numbers.** The "150+ organizations" and "enterprise production use" framing comes from a Linux Foundation press release. Governance and membership are authoritative there; the adoption framing is promotional, and independent commentators argue supporter counts substantially overstate real deployment.

## ACP — read the tiers carefully

Created by Zed, announced **27 August 2025** with Google's Gemini CLI as first integration.

- **First-party, real:** Zed; **JetBrains** — announced 6 Oct 2025, shipped in IDEs **2025.3**, ACP Agent Registry **Jan 2026**, own fork at `jetbrains/agent-client-protocol`. JetBrains is the *only* genuine second vendor implementation.
- **Community, not vendor:** VS Code (via `formulahendry/vscode-acp`), Neovim, Emacs, marimo, Obsidian, DeepChat, Tidewave. Some are untested in compatibility matrices.

**VS Code is not Microsoft support.** Issue `microsoft/vscode#265496` remains open; a Microsoft engineer said they don't plan to implement it. VS Code's agent mode is standardised on MCP.

**Naming collision:** always spell out *Agent Client Protocol*. "ACP" also means IBM/BeeAI's *Agent Communication Protocol*, which is archived and folded into A2A.

## `SKILL.md` frontmatter

Exactly six fields. Two required (`name`, `description`), four optional (`license`, `compatibility`, `metadata`, `allowed-tools`).

- `name` — max 64 chars, lowercase/numbers/hyphens, no leading/trailing/consecutive hyphens, **must match parent directory name**
- `description` — max 1024 chars, non-empty
- `compatibility` — max 500 chars
- `allowed-tools` — **Experimental**, support varies

Non-spec fields → `Unexpected key(s)` error on upload/packaging.

Validate in CI: `skills-ref validate ./my-skill`

> Spec-vs-implementation divergence: Claude Code itself does not enforce the spec strictly.
