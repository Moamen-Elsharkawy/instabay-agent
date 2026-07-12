# Approach & Research Note

What I explored before building, what I adopted, and what I rejected - with sources.

## Design thesis

An Instagram "marketing teammate" is a **closed-loop content operation**, not a chatbot: strategy -> creation -> quality gate -> human approval -> publish -> measure -> learn -> better creation. The failure modes worth engineering against are (1) off-brand or unnatural-Arabic output, (2) fake "it works" claims with no numbers, and (3) autonomous behavior a business can't trust. Every architectural choice below traces to one of those three.

## Adopted

**1. Orchestrated workflow with deterministic control flow, not an autonomous agent swarm.**
Anthropic's "Building Effective Agents" (Dec 2024) distinguishes *workflows* (LLMs orchestrated through predefined code paths) from *agents* (the LLM steers itself), and explicitly recommends simple, composable patterns over frameworks for well-understood tasks. Content operations are a well-understood pipeline, so each stage is a fixed n8n node graph, and the LLMs are specialists inside it: Brand Strategist, Creative Director, Brand Judge, Reviser, Performance Analyst.
Source: https://www.anthropic.com/engineering/building-effective-agents

**2. Evaluator-optimizer at two timescales.**
The same Anthropic taxonomy names the *evaluator-optimizer* pattern (one model generates, another evaluates against clear criteria, loop). It runs here twice: per-batch (Judge -> Reviser -> Re-Judge, exactly one revision pass) and cross-batch (engagement data -> Analyst directives -> next batch). The single revision pass follows the diminishing-returns finding of Self-Refine (Madaan et al. 2023, arXiv:2303.17651).

**3. LLM-as-judge with a different model family than the generator.**
Zheng et al. 2023 (arXiv:2306.05685) established LLM judges reach >80% human agreement but documented *self-enhancement bias* - judges prefer their own outputs. So the generator is Anthropic (`claude-haiku-4.5`) and the judge is OpenAI (`gpt-4o-mini`, temperature 0). Scoring uses a G-Eval-style anchored rubric with step-by-step reasoning before committing scores (Liu et al. 2023, arXiv:2303.16634). The judge is paired with a **deterministic lint** (always-on hashtags present, 7-11 tags, no banned tags, caption lengths, bilingual presence) so part of the brand-consistency benchmark is fully rule-based and reproducible.

**4. Reflexion-style verbal memory instead of fine-tuning or vector memory.**
Reflexion (Shinn et al. 2023, arXiv:2303.11366) showed verbal self-reflections stored as episodic memory improve later attempts. Here the Learnings tab IS that memory: the Analyst writes 3-5 evidence-cited directives per cycle ("keep/stop/try" on format/slot/pillar/caption-style/hashtags), and the next batch's Creative Director must apply them and cite which directive shaped each post (`rationale` column - making the loop auditable post by post).

**5. Native bilingual generation, never translate-after.**
Arabic captions are written FIRST, natively, in polished Egyptian colloquial with GCC readability - never translated from English. Benchmarks consistently show frontier general models outperform Arabic-specialist models (Jais, AceGPT, Fanar) on Arabic knowledge and dialect generation (ArabicMMLU, arXiv:2402.12840; 2025 Arabic-LLM survey, arXiv:2510.13430; Frontiers in AI 2025 cross-dialect study where GPT-4-class models led). The judge's `arabic_naturalness` rubric explicitly penalizes translation calques and stiff MSA - the failure mode of naive pipelines.

**6. Structured outputs everywhere.**
Every agent emits schema-validated JSON (n8n structured output parser, explicit JSON Schema, auto-fix repair pass). Constrained/validated output is now table stakes for reliable agent pipelines. Two hard lessons got engineered in (see assumptions doc): example-inferred schemas mark every field required (too brittle for creative output - use explicit schemas with a minimal `required` list), and an auto-fix pass can *hallucinate* plausible-but-wrong content when the raw output was truncated - so a deterministic **batch guard** (count + ID-pattern checks) fails loudly rather than letting a "repaired" batch through.

**7. Seeded synthetic engagement with hidden structure.**
See `engagement-simulator.md`. Constants grounded in hospitality Instagram benchmarks (Social Insider ~0.46% median hospitality ER; Dash Social travel ~2%+ strong; Sprout Social format effects, reels ~2.7% vs static ~1.3%). Ground truth lives only inside the simulator node - no prompt leakage - so the optimization uplift measures genuine discovery by the loop.

**8. Human-in-the-loop as a hard gate.**
Nothing publishes without explicit human approval (n8n-hosted approval form showing the full bilingual batch + judge scores; approve-all or request-changes-with-feedback, where human feedback outranks judge notes in the Reviser). Aligned with how LangChain's `social-media-agent` handles HITL.

## Rejected (and why)

- **Autonomous auto-posting agents** (e.g. `alsk1992/instagram-ai-agent`: fully autonomous posting/replies with anti-detection): impressive engineering, wrong trust model for a business account, and anti-detection tooling violates platform terms. This system uses only official APIs and keeps a human gate.
- **Multi-armed bandits / Thompson sampling** for slot/format selection (headline-testing bandits, arXiv:1908.06256; ad-experiment bandits, arXiv:1907.02178): statistically principled but wrong for n=7 posts/week - the arms barely get one pull each per cycle. Verbal directives are more sample-efficient and human-inspectable at this volume. Revisit at 10x posting volume or when running paid ads.
- **Arabic-specialist models** (Jais, AceGPT, Fanar): benchmark evidence above; frontier general models write better Egyptian-dialect marketing copy.
- **Meta's official MCP server** for publishing: it is ads-only (Meta Ads AI Connectors, open beta April 2026) - it cannot publish organic content. Organic publishing integrates the Graph API directly ("Instagram API with Instagram Login" product: no Facebook Page needed, dev-mode publishing without App Review - see `go-live-path.md`).
- **Community n8n Instagram nodes**: unofficial, small install bases, another dependency to trust. Plain HTTP requests against the documented Graph API are transparent and swappable.
- **A rubric-builder agent** (as in some judge-loop templates): a static anchored rubric embedded in the judge prompt is more reproducible across batches - a moving rubric would corrupt batch-over-batch benchmark comparability.
- **Fine-tuning / RAG memory for brand voice**: the brand constitution fits comfortably in context (context engineering beats retrieval at this scale); a vector store would add infrastructure without adding quality.

## Model & cost choices (ROI note)

Default-to-cheap policy with one deliberate paid choice: `anthropic/claude-haiku-4.5` (generator, ~$1/$5 per M via OpenRouter) over free-tier models, because free endpoints rate-limit unpredictably mid-run, and Gemini 2.5's default hidden "thinking" tokens consumed the output budget and truncated 7-post batches (observed during build - documented in assumptions). A full generate+judge+revise batch costs a few cents; per-batch cost and latency are measured live from the OpenRouter account-usage delta and logged to the RunLog tab (see the README benchmark table).
