# LLM application security

**Defensive design, not attack craft.** This file is about how you build an LLM application or agent so a manipulated model can't do much damage. It stays at the architecture level and contains no working attack payloads.

It sits next to three other files. [agent-protocols.md](agent-protocols.md) covers MCP-specific security: token passthrough and the confused-deputy proxy attack (questions 4-5), and the *Parasites in the Toolchain* survey of MCP servers (question 6). This file doesn't repeat any of that. [agent-architecture.md](agent-architecture.md) covers how agents are structured, and [retrieval-and-rag.md](retrieval-and-rag.md) covers the retrieval pipeline that poisoning targets.

The framing interviewers reward: **prompt injection is an architecture problem, not a prompt problem.** Candidates who answer with "a better system prompt" or "add a classifier" are describing mitigations, not controls.

---

### 1. What is the difference between direct and indirect prompt injection, and why does the indirect kind matter more?

`verified 3-0`

<details>
<summary>Answer</summary>

**Direct injection:** the user types instructions meant to override the application's own instructions. The attacker is the person at the keyboard, so the damage is mostly limited to what that user could already reach. It's a policy problem (jailbreaks, system-prompt leakage), and usually a smaller one.

**Indirect injection:** the instructions are hidden in content the model *reads*: a web page, an email, a retrieved document, a tool output, a code comment. The attacker never talks to your system. They only have to put text somewhere your system will ingest it. The model then acts **with the victim's privileges**.

That is why indirect matters more. The foundational paper is **Greshake et al., *Not what you've signed up for: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection*** (arXiv [2302.12173](https://arxiv.org/abs/2302.12173), 2023). Its core observation is that LLM-integrated applications *"blur the line between data and instructions,"* so retrieved text effectively becomes code. The paper's threat taxonomy covers data theft, worming (self-propagation), information-ecosystem contamination and API manipulation, and it demonstrated these against Bing Chat and code-completion tools.

The line worth having ready: **with indirect injection the threat model changes from "malicious user" to "anyone who can write text your agent will eventually read."**

</details>

---

### 2. Why is there no complete fix for prompt injection? What does a layered defense look like?

`verified 3-0`

<details>
<summary>Answer</summary>

**Why it can't be fully fixed:** the model gets instructions and data in the same token stream and has no hard boundary between them. Every defense that works *inside* the model (delimiters, "ignore instructions in the documents below", fine-tuning, classifiers) is probabilistic. An attacker can try as many times as they like, so a 95%-effective filter is not a security control. Simon Willison puts it bluntly: *"95% is very much a failing grade"* ([The lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/), June 2025).

**The layered answer** is what prep guides agree a strong candidate gives, and it lines up with OWASP LLM01:

1. **Treat all input *and* output as untrusted**, including your own model's output.
2. **Separate system, user and retrieved content** with clear delimiters. This helps only partly: it lowers the attack success rate but doesn't create a boundary.
3. **Sanitize tool outputs before they go back into context.** Also partial.
4. **Give every tool least privilege and run it in a sandbox.** This is where most of the real protection comes from (Q6).
5. **Filter or moderate outputs**, especially anything that renders or makes network requests (Q3).
6. **Require human approval before irreversible actions** (Q7).
7. **Red-team continuously.**

The trade-off to say out loud: **layers 2, 3 and 5 reduce how *often* attacks succeed, while layers 4 and 6 limit what an attack can *do*.** When you have a limited security budget, spend it on limiting what an attack can do.

</details>

---

### 3. Explain the "lethal trifecta." How does it change how you review an agent design?

`single-source`

<details>
<summary>Answer</summary>

Willison's framing ([June 16, 2025](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)): an agent is exposed to data theft when it combines

1. **access to private data**,
2. **exposure to untrusted content**, and
3. **the ability to communicate externally.**

Any two are manageable. All three together means injected text in (2) can read (1) and send it out through (3).

**Why it's useful in review:** it turns a question you can't answer ("is the model robust to injection?") into one you can ("which of these three capabilities can we remove *for this task*?"). Removing any single leg breaks the chain, and removing a leg is deterministic.

**The leg people forget is exfiltration, because it doesn't look like a tool.** The classic example is **markdown image rendering**. If the model's output is rendered as markdown and can contain an image whose URL the model chose, the client fetches that URL automatically. Anything the model put in the query string leaves with no click needed. Willison has documented this class of bug across several products, including GitLab Duo, Superhuman and Microsoft 365 Copilot ([markdown-exfiltration tag](https://simonwillison.net/tags/markdown-exfiltration/)). The fixes were image-domain allow-lists, CSP, or not rendering untrusted images at all. One instance got past a CSP through an expired domain that was still on the allow-list. Other outbound channels to audit the same way: link unfurling, web-fetch tools, email/ticket creation, and DNS lookups from sandboxes.

Senior point: **a "read-only" agent with a markdown renderer is not read-only.**

</details>

---

### 4. Walk through the OWASP Top 10 for LLM Applications. Which items are really about architecture?

`single-source`

<details>
<summary>Answer</summary>

The 2025 list ([genai.owasp.org](https://genai.owasp.org/llm-top-10/)):

| ID | Risk |
|---|---|
| LLM01 | Prompt Injection |
| LLM02 | Sensitive Information Disclosure |
| LLM03 | Supply Chain |
| LLM04 | Data and Model Poisoning |
| LLM05 | Improper Output Handling |
| LLM06 | Excessive Agency |
| LLM07 | System Prompt Leakage |
| LLM08 | Vector and Embedding Weaknesses |
| LLM09 | Misinformation |
| LLM10 | Unbounded Consumption |

**Don't recite it. Group it.** The architecture-level items are **LLM01 + LLM05 + LLM06**. Injection is what goes in, improper output handling is the model's output reaching a renderer, shell or SQL unchecked, and excessive agency is how far the damage can spread. Most real incidents involve all three together, which is the lethal trifecta in OWASP's terms.

**LLM04 and LLM08 are the RAG items.** Poisoning your retrieval corpus is indirect injection that *persists*: one planted document keeps getting served to every user whose query retrieves it. Design answers: record provenance and trust level per chunk, enforce access control *at retrieval time* rather than after generation (otherwise the vector store leaks across tenants), restrict who can write to indexed sources, and keep untrusted chunks out of the context of any step that can take actions. For pipeline details, see [retrieval-and-rag.md](retrieval-and-rag.md).

**LLM07 is a trap question.** The answer is that a system prompt should never contain secrets or be the thing enforcing authorization. If leaking it hurts you, the design is already wrong.

</details>

---

### 5. What design patterns give *provable* resistance to injection, and what do they cost?

`single-source`

<details>
<summary>Answer</summary>

The principle, from **Beurer-Kellner et al., *Design Patterns for Securing LLM Agents against Prompt Injections*** (arXiv [2506.08837](https://arxiv.org/abs/2506.08837), June 2025, authors from institutions including IBM, Invariant Labs, ETH Zurich, Google, Microsoft): once an agent has read untrusted input, it must be constrained so that input *cannot* trigger consequential actions.

The six patterns, roughly from most to least restrictive:

- **Action-selector:** the model picks from fixed actions and never sees tool responses.
- **Plan-then-execute:** the tool-call plan is fixed *before* any untrusted content is read.
- **LLM map-reduce:** isolated sub-agents each handle one untrusted item, and their results are combined in a constrained way.
- **Dual LLM:** a privileged LLM plans. A quarantined LLM reads untrusted content and returns only symbolic variables that the privileged one never reads.
- **Code-then-execute:** the privileged LLM writes a program in a sandboxed DSL.
- **Context minimization:** strip content that is no longer needed (e.g. the original user prompt) before later steps.

**CaMeL** (Debenedetti, Shumailov, Carlini et al., Google DeepMind; *Defeating Prompt Injections by Design*, arXiv [2503.18813](https://arxiv.org/abs/2503.18813)) is the most developed version of code-then-execute. It extracts control flow and data flow from the *trusted* query so that untrusted data "can never impact the program flow", and it attaches **capabilities** to values so policies block exfiltration at the point a tool is called. On AgentDojo it solves **77% of tasks with provable security, against 84% undefended**.

**The trade-off is the whole answer:** you give up 7 points of utility in exchange for a guarantee instead of a probability, plus the engineering cost of writing policies. These patterns also fit poorly with open-ended "read my inbox and do whatever it says" agents, and that's intended. The honest conclusion is that some product shapes can't be secured, and the designer's job is to say so.

</details>

---

### 6. How do you design tools and credentials for least privilege in an agent?

See also [agent-architecture.md question 5](agent-architecture.md#5-whats-your-credential-hygiene-for-an-agent-that-touches-real-systems) for the harness-level version of this question.

`single-source`

<details>
<summary>Answer</summary>

**Tool design:**

- **Narrow verbs, not general ones.** `create_ticket(project, title, body)` rather than `http_request(url, method, body)`. A generic tool is an exfiltration channel with a friendly name.
- **Put policy in code at a single chokepoint, not in the prompt.** Example: the open-source [webagent](https://github.com/TheAgent-net/webagent) wraps every tool, including ones injected by the host, in `action.Guard`, which runs the guardrail *before* execution. In its own words, action safety is *"code-enforced, not prompt-enforced."*
- **Sandbox untrusted execution behind a serialized boundary.** [Vercel Run](https://vercel.com/blog/introducing-run) evaluates JS/type-stripped TS in a fresh QuickJS context inside a worker thread, with no direct route to Node.js or the network. Only explicitly exposed `hostFunctions` are reachable, and calls to them cross the boundary through serialization. The design idea to take away is that the sandbox can only do what you deliberately passed across.

**Credential scoping:**

- **Separate what the agent owns from what the user owns.** LangSmith's [Connections](https://www.langchain.com/blog/connections-managed-credentials-and-per-caller-identity-for-managed-deep-agents) makes this explicit. An *agent-owned* credential belongs to the deployment and every caller shares it (fine for web search or a pricing feed). A *user-owned* credential resolves per caller at run time, so a GitHub issue is opened *by the person who asked*, not by a service account. Per-caller identity is what gives you an audit trail and stops one user from acting through the agent with another user's access.
- Short-lived, scoped tokens; never forward the user's token to a downstream service. That's the MCP token-passthrough rule, see [agent-protocols.md question 4](agent-protocols.md#4-why-is-token-passthrough-forbidden-in-mcp-and-what-must-a-server-do-instead).
- **Keep an inventory of what can execute or reach the network.** You can't scope what you haven't listed. [geiger-scan](https://github.com/Atomburstofficial/geiger) is one read-only example: it lists agents, MCP servers, plugins and extensions on a machine and labels each with capabilities such as `[EXECUTES]`, `[HOLDS-SECRETS]` and `[BROAD-FILESYSTEM]`, and it has a `--diff` mode to catch drift.

</details>

---

### 7. When do you require a human in the loop, and how do you stop the approval step itself from being attacked?

`single-source`

<details>
<summary>Answer</summary>

**When:** before irreversible actions or actions visible outside the system: sending, purchasing, deleting, publishing, and changing permissions. Not before every action. Approval fatigue turns a human checkpoint into a button people click without reading, which is worse than having no checkpoint, because it makes the system look safer than it is.

**The design problem most candidates miss:** if the approval request appears *in the conversation*, injected text can imitate it or frame the request misleadingly. The approval channel has to be something the model can't write to.

The most complete published example is Meta's harness for its Muse agent, as reported in [The Batch, issue 371](https://www.deeplearning.ai/the-batch/issue-371):

- A single component (Sentinel) decides whether an action is allowed. It checks every outbound request against the permissions the user set.
- Approvals appear as a **system dialog, not a chat message**, so injected text can't fabricate one.
- Each approval is **bound to one connector or destination and one purpose**. Purchases and emails always require confirmation.
- Underneath that: data from outside sources is labeled untrusted when it enters context, and a separately trained classifier ensemble runs outside the agent's runtime.

**The senior caveat:** Meta evaluated injection resistance on an unpublished dataset and published no classifier accuracy. It offers a bug bounty instead: **$130,000 for a successful prompt injection**, with $300,000 as the overall maximum payout for a valid report, not the injection-specific figure. The architecture is well designed but has no public measurement. Say which of those two you'd rely on.

</details>

---

### 8. What are guardrail classifiers good for, and where do they fail?

`single-source`

<details>
<summary>Answer</summary>

**Good for:** reducing volume. They catch unsophisticated and known injection patterns cheaply, they flag suspicious tool outputs for logging and review, and they are useful as *telemetry*. A spike in classifier hits tells you someone is probing.

**Where they fail:**

- **They are probabilistic against an adversary who can retry.** A detection rate that sounds high is a failing grade in security terms ([Willison](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)). The attacker only needs the one attempt that gets through.
- **They check content, and injections can be rephrased indefinitely.** Controls based on *behaviour* (is this tool call something the user's task required?) hold up better than controls based on *content*.
- **They can be attacked themselves** if they run somewhere the agent can influence. That's why Meta runs its classifiers outside the runtime (Q7).
- **Checking your own work is not verification.** The same idea applies to security review: Cloudflare's [security-audit-skill](https://github.com/cloudflare/security-audit-skill) states that *"the agent that checks a finding is never the agent that found it"* and gives each candidate finding to a fresh verifier that tries to disprove it. That is the same principle as using a separately trained classifier rather than asking the acting model whether it was injected.

**How to position them:** classifiers sit *in front of* deterministic controls, never *in place of* them. If removing the classifier would make the system unsafe, the architecture isn't doing its job.

</details>
