# Assumptions & Trade-offs

The brief says: make reasonable assumptions, note them, keep moving. Here they are, plus the trade-offs taken inside the ~5-6h time-box and the real failures hit during the build (kept because they shaped the design).

## Assumptions

1. **Fictional resort, real pipeline.** Insta Bay has no real Instagram account, so `publish_mode=dry_run` is the default: the Publisher logs the exact Graph API call sequence per post instead of sending it. The live branch is fully built and swappable (`go-live-path.md`).
2. **Follower base = 12,000** (Config) for the engagement simulator - a plausible mid-size resort account. Changing it rescales reach, not the loop's logic.
3. **Weekly batch of 7 posts: 1 reel + 1 carousel + 5 images** - defined by the generated brand constitution's cadence, matched to the brief's "posts inconsistently" problem.
4. **The approver is one person** (the marketing owner) reached by email + a hosted approval form. Multi-reviewer flows are out of scope.
5. **English docs, bilingual content.** The submission is in English; all guest-facing content is Egyptian Arabic + English, Arabic written natively first.
6. **Reel deliverable = concept + storyboard**, not a rendered video. The live reel path expects a hosted MP4 produced from the storyboard.

## Trade-offs (build vs stub)

| Decision | Chose | Why |
|---|---|---|
| Orchestration | n8n workflows | Deterministic control flow, visual auditability for a non-technical client, production-ready scheduling/retries out of the box |
| State store | One Google Sheet, 7 tabs | Human-visible and human-editable (the approver can read the calendar), zero infra; a DB would add ops burden without adding capability at this scale |
| Revision depth | Exactly 1 judge-triggered revision pass per batch | Self-Refine shows diminishing returns; unlimited loops burn tokens and can oscillate. Human "request changes" can loop as many times as the human wants |
| Judge scope | Batch-level calls (all 7 posts in one judge call) | 4 LLM calls per batch instead of ~20; per-item routing in n8n is where builds die |
| Carousel live publish | Stubbed (dry-run documents the child-container sequence) | Needs a container-per-slide loop; documented, ~1h to add |
| Comment/DM handling | Out of scope | Different risk class (real-time guest interaction); noted in if-this-were-real |

## Real failures hit during the build (and what they changed)

These are left in deliberately - the fixes are part of the architecture now.

1. **Example-inferred JSON schemas are brittle.** n8n's structured output parser in "generate from JSON example" mode marks *every* field in the example as required (including nested placeholder fields). The first brand-generation run failed on it. Fix: explicit JSON Schema mode everywhere, with a minimal `required` list.
2. **Auto-fix can hallucinate.** When the generator's output was truncated, the parser's auto-fix pass "repaired" it into a schema-valid but completely off-brief batch (a generic AI-topics post with wrong dates). Fix: a deterministic **batch guard** after generation - post count must equal the plan, post IDs must match the `b{batch}-p{n}` pattern - that fails the run loudly instead of letting plausible garbage through. Lesson: schema compliance is not semantic correctness.
3. **Hidden reasoning tokens truncate long outputs.** Gemini 2.5 Flash's default "thinking" consumed the output budget on 7-post bilingual batches. Fix: switched the generator to `anthropic/claude-haiku-4.5` (no hidden thinking, strong Arabic) and set explicit `maxTokens`. This also strengthened the judge design: generator (Anthropic) and judge (OpenAI) are now different vendors.
4. **OAuth credentials die silently.** The instance's Gmail credential had expired; a send-and-wait approval email was a single point of failure. Fix: the approval gate is now an **n8n-hosted form** (works regardless of email), its URL is logged to the RunLog sheet every run, and the Gmail notification is a non-fatal extra (`onError: continue`). The human gate no longer depends on any third-party credential.
5. **The revision agent can return a partial batch.** An early revision pass returned only the changed post. Fix: the merge step overlays revised posts onto the full original batch by `post_id` - posts can be changed, never lost.

## Security notes

- No secrets live in the repo or the sheet except the demo convenience of `ig_access_token` in Config for live mode - flagged in go-live-path.md to be moved to an n8n credential before real use.
- Webhook endpoints used for testing are unauthenticated demo endpoints; production would add header auth (n8n supports it natively on webhook and wait nodes).
- The approval form URL contains an n8n-signed token; only holders of the link can approve.
