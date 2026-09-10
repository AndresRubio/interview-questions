# Agent protocols

MCP, A2A, Agent Client Protocol, Agent Skills. Almost everything here is `verified 3-0`, and almost all of it changed in 2026 — which makes it unusually high-value interview material, because most published tutorials are still describing the old semantics.

> **Perishable.** MCP revision `2026-07-28` was ~6 weeks old when these notes were compiled. Re-check the [changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog) before quoting version-specific behaviour.

---

### 1. MCP went stateless in the 2026-07-28 revision. What broke, and how would you migrate a server?

`verified 3-0`

<details>
<summary>Answer</summary>

Three removals, all breaking changes from `2025-11-25`:

- **The `initialize` / `notifications/initialized` handshake is gone.** Protocol version and client capabilities now travel per-request in `_meta`, as `io.modelcontextprotocol/protocolVersion` and `io.modelcontextprotocol/clientCapabilities`, with `UnsupportedProtocolVersionError` on mismatch (SEP-2575). This is verifiable mechanically: the 2026-07-28 `schema.json` contains zero occurrences of `initialize` — the method is absent, not deprecated.
- **Protocol-level sessions and the `Mcp-Session-Id` header are removed** from Streamable HTTP. List endpoints (`tools/list`, `resources/list`, `prompts/list`) no longer vary per connection. Cross-call state uses explicit server-minted handles passed as ordinary tool arguments (SEP-2567).
- **SSE resumability is removed** — no `Last-Event-ID`, no SSE event IDs. A broken stream MUST be re-issued as a new request with a new request ID.

Migration: anything previously held in session state becomes an explicit handle the server mints and the client passes back as a normal argument.

The *why* is the good part of the answer: statelessness means servers scale horizontally behind a load balancer with no sticky sessions. The old `Mcp-Session-Id` actively fought load balancers.

Related: the "session hijacking" attack class was renamed "state handle hijacking" and scoped to `2025-11-25` and earlier.

</details>

---

### 2. Sampling and elicitation used to be server-initiated. They aren't any more. What replaced them?

`verified 3-0`

<details>
<summary>Answer</summary>

The **Multi Round-Trip Requests (MRTR)** pattern.

`roots/list`, `sampling/createMessage` and `elicitation/create` MUST now go through it. The server returns an `InputRequiredResult` carrying `resultType: "input_required"` and an `inputRequests` field; the client then retries the original request with `inputResponses` attached — and the spec makes it a MUST that **the JSON-RPC id differs between the initial request and the retry**.

All results now carry a required `resultType` of either `complete` or `input_required`. Clients MUST treat a missing `resultType` from an earlier-protocol server as `complete` (SEP-2322).

The spec is blunt about it: *"The previous pattern of server-initiated requests is no longer supported. This is a breaking change."*

Worth knowing: `inputRequests` is itself optional in the schema, so a defensive client handles its absence.

</details>

---

### 3. What is deprecated in MCP right now, and on what timeline?

`verified 3-0`

<details>
<summary>Answer</summary>

MCP adopted a formal **Active / Deprecated / Removed lifecycle** with a minimum twelve-month deprecation window and a public registry (SEP-2596).

**Deprecated as of 2026-07-28** (SEP-2577), earliest removal being the first revision on or after **2027-07-28**:

| Feature | Suggested migration |
|---|---|
| Roots | Tool parameters, resource URIs, or server configuration |
| Sampling | Direct LLM provider APIs |
| Logging | stderr, or OpenTelemetry |

These remain fully functional during the window, but new implementations should not add support for them.

**Removed outright**, not deprecated: `logging/setLevel` and `notifications/roots/list_changed` (SEP-2575). Log level is now per-request via `io.modelcontextprotocol/logLevel` in `_meta`.

The detail that shows you actually read the registry: **HTTP+SSE does not get the twelve-month clock.** It was reclassified as Deprecated, but its earliest removal is *three months after SEP-2596 reaches Final* — a much shorter fuse than everything else.

</details>

---

### 4. Why is token passthrough forbidden in MCP, and what must a server do instead?

`verified 3-0`

<details>
<summary>Answer</summary>

Because accepting a token that wasn't issued for you breaks the OAuth audience boundary. You become an oracle that will act on any token a caller happens to hold — and the token's real issuer has no idea you exist.

The normative rules:

- A server **MUST validate that access tokens were issued specifically for it as the intended audience**, per **RFC 8707 §2**.
- A server **MUST NOT accept or transit any other tokens.**

The word "transit" is what forbids the second half — forwarding a client's token unmodified to a downstream API.

The spec's own diagnosis: when a server doesn't verify tokens were intended for it, it may accept tokens issued for other services, and *"this breaks a fundamental OAuth security boundary."*

In practice: validate `aud` against your own resource identifier on every request, and when you call downstream services use your own credentials or a proper token exchange — never the caller's token.

</details>

---

### 5. Describe the confused-deputy attack against an MCP proxy server, and its required mitigations.

`verified 3-0`

<details>
<summary>Answer</summary>

It requires **four conditions simultaneously**:

1. A static third-party client ID
2. Dynamic client registration for MCP clients
3. A third-party consent cookie set after the first authorization
4. No per-client consent at the proxy

With all four present, a second client silently inherits the first client's consent — the proxy acts as a deputy confused about whose authority it's exercising.

**Mandated mitigations:**

- Maintain a **registry of approved `client_id` values per user**, and check it **before** initiating the third-party authorization flow
- Validate `redirect_uri` by **exact string matching** — explicitly not pattern matching or wildcards
- Make `state` values **single-use** (delete after validation) with **short expiry**, e.g. 10 minutes
- Do not set a consent cookie until the user has approved the MCP server's *own* consent screen

</details>

---

### 6. How do you actually secure an MCP deployment? Don't just recite the spec.

`verified 3-0`

<details>
<summary>Answer</summary>

Start with the root cause, which is architectural rather than a bug in any one server: **MCP provides neither context-tool isolation nor least-privilege enforcement**, so instructions injected into retrieved content propagate unchecked into sensitive tool calls.

Then the empirical backing. *Parasites in the Toolchain* (Zhao et al., SJTU / CHAITIN / HKUST), **IEEE S&P 2026, pp. 138–155** — peer-reviewed, not just a preprint — surveyed **1,360 servers and 12,230 tools** from PulseMCP, MCP Market and Awesome MCP Servers:

- **1,062 tools (8.7%)** and **370 servers (27.2%)** expose capabilities composable into privacy-exfiltration toolchains
- Of **10 toolchains** built from real popular servers (e.g. `fetch` → `filesystem read_file` → `gmail send_mail`), **9 produced verified exfiltration** in at least one of 10 trials in Cursor; three succeeded 6–8 times out of 10 **without adversarial prompt tuning**

Proposed defences: context-tool isolation, privilege minimisation, cross-tool invocation auditing. Add human approval on egress-capable tools and audience validation on every token.

**Volunteer the limits — this is the part that reads as senior:**

- The census is a mid-2025 snapshot of three curated directories, precision-corrected to ~7.4%
- It measures *composable affordances*, not individually exploitable tools
- End-to-end results were obtained with **human approval deliberately disabled**
- Results are strongly **model-dependent** — GPT-5 and Claude 4.5 Sonnet resisted all attacks
- The three defences are stated design principles, not a benchmarked system

</details>

---

### 7. MCP, A2A, ACP, Agent Skills — how do they compose? Where do they compete?

`verified 3-0`

<details>
<summary>Answer</summary>

Mostly they layer. The discourse overstates the competition.

- **MCP** is vertical: an agent reaching tools and data.
- **A2A** is horizontal: agents coordinating **across organisational and trust boundaries**.
- **ACP** (Agent Client Protocol) is a third axis: **editor ↔ agent**, so any agent runs inside any editor.
- **Agent Skills / `SKILL.md`** is packaging — portable capabilities an agent loads.

The MCP/A2A split is officially framed as complementary, and that framing is now structural rather than diplomatic: both sit under the **Agentic AI Foundation**, a Linux Foundation directed fund. Anthropic donated MCP on **2025-12-09**; A2A joined on **2026-08-17**, alongside goose, AGENTS.md and agentgateway.

The Linux Foundation's own wording: *A2A defines how agents communicate and coordinate with each other across organizational boundaries, while MCP defines how agents connect to internal tools and data sources.*

The genuine competition is narrower than people think: **ACP vs MCP at the editor layer**, where Microsoft has declined ACP in VS Code in favour of MCP.

</details>

---

### 8. What does the `SKILL.md` specification actually constrain? Be precise.

`verified 3-0`

<details>
<summary>Answer</summary>

Exactly **six** YAML frontmatter fields — two required, four optional:

| Field | Required | Constraint |
|---|---|---|
| `name` | yes | max 64 chars; lowercase letters, numbers, hyphens only; no leading, trailing or consecutive hyphens; **must match the parent directory name** |
| `description` | yes | max 1024 chars, non-empty |
| `license` | no | — |
| `compatibility` | no | max 500 chars |
| `metadata` | no | — |
| `allowed-tools` | no | marked **Experimental**; support varies between implementations |

Non-spec fields produce an `Unexpected key(s)` error on upload or packaging.

There's an official reference validator — `skills-ref validate ./my-skill` — in the `agentskills/agentskills` repo, which makes spec conformance a CI check rather than a claim.

**Adoption is genuinely multi-vendor**, not Anthropic-only: the client showcase lists **46 implementing products**, including OpenAI (ChatGPT and Codex), GitHub Copilot, VS Code, Google Gemini CLI, Cursor, Mistral, JetBrains Junie, Amp, OpenHands and Goose. Verified at vendors' own docs, including direct commercial rivals.

Nuance worth knowing: **Claude Code itself does not enforce the spec strictly** — a spec-versus-implementation divergence.

</details>

---

### 9. What's wrong with saying "VS Code supports ACP"?

`verified 3-0`

<details>
<summary>Answer</summary>

It's a community extension, not Microsoft. `microsoft/vscode` issue **#265496** is still open, and a Microsoft engineer stated they don't plan to implement it. VS Code's agent mode is standardised on **MCP**.

The "Editors on ACP" list is Zed's own promotional adoption directory and mixes tiers badly. Sorting it out:

- **First-party, real:** Zed (creator), and **JetBrains** — announced 6 Oct 2025, shipped in IDEs **2025.3**, ACP Agent Registry launched **Jan 2026**, with their own fork at `jetbrains/agent-client-protocol`. JetBrains is the only genuine second vendor implementation.
- **Community:** VS Code (via `formulahendry/vscode-acp`), Neovim, Emacs, marimo, Obsidian, DeepChat, Tidewave — with some shown as untested in compatibility matrices.

**Also always spell out "Agent Client Protocol".** "ACP" collides with IBM/BeeAI's *Agent Communication Protocol*, which is archived and folded into A2A. The bare acronym is ambiguous.

Governance detail: ACP is Apache-2.0 with no CLA, created by Zed (announced **27 Aug 2025**, Gemini CLI as first integration), now under an explicitly **interim** model co-governed by Zed and JetBrains with two Lead Core Maintainers acting as BDFLs. A transition to independent foundation governance is intended but **undated** — and notably, ACP is *not* among the Agentic AI Foundation contributions.

</details>
