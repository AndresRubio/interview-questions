# Refuted claims

These are plausible-sounding statements that **failed adversarial verification** — independent reviewers were asked to refute each claim, and these lost by majority or unanimous vote.

They're kept rather than deleted because several are things you'd plausibly say in an interview, and several circulate widely. Knowing what *doesn't* hold up is worth as much as knowing what does.

| Claim | Vote |
|---|---|
| That the canonical agent structure is a three-phase loop: **gather context → take action → verify work** | `0-3` |
| That verification methods rank in robustness as: deterministic rules-based → visual feedback → LLM-as-judge (least robust) | `1-2` |
| That MCP `2026-07-28` introduced **Client ID Metadata Documents (CIMD)** — URL-based client IDs fetched by the authorization server — with associated SSRF and localhost-impersonation risks | `0-3` |
| That A2A v1.0 added **Signed Agent Cards** for cryptographic identity verification | `0-3` |
| That Agent Skills is framed as a vendor-neutral standard originally created by Anthropic and adopted beyond its originator | `1-2` |
| That `SKILL.md` specifies a three-stage progressive-disclosure loading model with concrete budgets — name+description ~100 tokens at startup, body under 5,000 tokens and 500 lines on activation | `0-3` |
| That Anthropic cache reads cost 0.1× base input with break-even at ~1.4 reads per write; that OpenAI caches automatically at 50% off with a 1,024-token minimum *(as stated by vendor blogs)* | `0-3` |
| That semantic caching serves 20–45% of production traffic, a cosine threshold ≥0.85 is the right default, and it cuts cost 40–70% (850ms → 120ms) | `0-3` |
| That hybrid model routing cuts LLM usage 37–46% and frontier models cost 20–30× more per token | `0-3` |
| That Claude Code's changelog records allow-list wildcards matching beyond their intended scope, or an OAuth refresh forcing an hourly prompt-cache miss | not found |

## Notes

The Agent Skills entry is subtle. **Multi-vendor adoption itself is well-supported** (`verified 3-0`) — 46 implementing clients, confirmed at vendors' own documentation including direct Anthropic competitors. What failed verification was the specific *framing* of origin and neutrality. Cite the adoption, not the origin story.

The progressive-disclosure entry is the one most likely to trip you up, because the *concept* of progressive disclosure in skills is real and widely discussed. What the specification does **not** do is mandate those particular token and line budgets. Describe the behaviour, don't quote numbers.

The caching-price entries need care. What failed was the **vendor-blog sourcing**: reviewers could not confirm those figures from the blogs cited. The current official Anthropic docs, fetched 2026-09-25, do show cache reads at roughly a 90% discount. Quote the provider's own pricing page with its date, never a third-party summary. [cost-and-caching.md](../questions/cost-and-caching.md) does this.

The Claude Code changelog entries were supplied as leads and not found on inspection. The only related entry (2.1.282) fixes the opposite behaviour: Bash rules with a mid-pattern `:*` were being skipped.

## Method

Each claim was independently reviewed by three agents instructed to find contradicting evidence. Two or more refutations killed the claim. Claims that survived 3-0 are marked as such throughout these notes; a 2-1 survival is flagged so it gets attributed rather than asserted.
