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

## Notes

The Agent Skills entry is subtle. **Multi-vendor adoption itself is well-supported** (`verified 3-0`) — 46 implementing clients, confirmed at vendors' own documentation including direct Anthropic competitors. What failed verification was the specific *framing* of origin and neutrality. Cite the adoption, not the origin story.

The progressive-disclosure entry is the one most likely to trip you up, because the *concept* of progressive disclosure in skills is real and widely discussed. What the specification does **not** do is mandate those particular token and line budgets. Describe the behaviour, don't quote numbers.

## Method

Each claim was independently reviewed by three agents instructed to find contradicting evidence. Two or more refutations killed the claim. Claims that survived 3-0 are marked as such throughout these notes; a 2-1 survival is flagged so it gets attributed rather than asserted.
