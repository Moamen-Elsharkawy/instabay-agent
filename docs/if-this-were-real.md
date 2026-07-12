# If This Were Real

What I would do next, in priority order, if Insta Bay were a paying client and this moved from challenge to production.

## Week 1 - go live safely

1. **Real publishing**: run the go-live path (`go-live-path.md`) - Business account, Meta app, Instagram-Login scopes, long-lived token in an n8n credential, flip `publish_mode=live`. Publish week 1 with double approval (form + a final "it's live" check on the first 2 posts).
2. **Real measurement**: switch the Analyst to the live insights branch; keep the simulator as a regression sandbox for testing directive logic without touching the account.
3. **Complete the carousel live path** (child-container loop) and a **token-refresh cron** (long-lived tokens expire in 60 days).
4. **Real visuals**: replace generated placeholder images with the resort's own photo library (visual pillar of the brand). The generated `image_prompt` doubles as a shot brief for the photographer.

## Month 1 - make the loop trustworthy

5. **Human anchor for the judge**: monthly, a native-speaker rates a sample of captions on the same rubric; compute judge-human agreement and recalibrate rubric anchors if drift appears (the known limit of LLM-as-judge).
6. **Slot/format experimentation discipline**: once real data flows, directives get promoted to "confirmed" only after 2+ batches of consistent signal; add a holdout slot per week permanently.
7. **Alerting**: a failed run or a >24h-unapproved batch pings the owner (Telegram/WhatsApp), so posts never silently stop - the failure mode the brief describes ("posts inconsistently") must not reappear via a stuck pipeline.
8. **Comment triage (read-only first)**: ingest comments via the API, classify (question/complaint/lead), and draft suggested replies for human sending - no auto-replies.

## Quarter 1 - scale the value

9. **Reels production line**: storyboard (already generated) -> template-based video assembly -> hosted MP4 -> the already-built 3-step reel publish path. Reels are the highest-reach format; this is the biggest single ROI unlock.
10. **Stories + DM funnel**: stories API support, plus a booking-inquiry DM flow into the resort's CRM with human handoff.
11. **Multi-account / multi-property**: the whole system is parameterized by one spreadsheet + one Config; a second property is a second sheet and cron set. This is where per-client dashboards (Looker Studio on the same sheet) come in.
12. **Paid amplification**: promote proven organic winners via the Marketing API - and this is where Meta's official ads MCP becomes relevant.

## Honest cost model at scale

Per weekly batch today: a few cents of LLM spend and ~3-4 minutes of wall-clock, with one human touchpoint (the approval). At 4 properties x 7 posts/week the marginal cost stays under $1/month of LLM spend; the binding constraint becomes the human approval and photography pipeline, not the AI.
