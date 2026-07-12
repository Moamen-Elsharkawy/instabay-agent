# Synthetic Engagement Simulator

The challenge allows a synthetic engagement dataset. This system uses a **seeded, deterministic simulator** (a Code node inside the Analyst workflow) instead of random numbers or a static CSV, for three reasons:

1. **Reproducible benchmarks.** Same posts in, same numbers out - re-running the Analyst produces byte-identical Metrics rows, so every number in the README can be recomputed by anyone who imports the workflows.
2. **A real signal to optimize against.** The simulator encodes documented, industry-grounded structure (formats, time slots, and topics genuinely differ), so the improvement loop has something true to discover - the "before/after uplift" measures whether the loop actually finds and exploits the structure.
3. **No leakage.** The ground-truth multipliers live ONLY inside the simulator's Code node. No agent prompt ever sees them. The Creative Director and Performance Analyst can only learn the structure the way a real marketer would: by observing per-post results.

## The model

For each published post (seeded per `post_id`, FNV-1a hash -> mulberry32 PRNG -> Box-Muller gaussian noise, sigma = 0.12):

```
reach      = follower_base                     (Config, default 12,000)
           x format_reach                      reel 0.45 | carousel 0.24 | image 0.20
           x slot_multiplier                   hour >= 21: 1.25 | >= 19: 1.15 | >= 15: 1.00 | else 0.90
           x pillar_affinity                   booking/offers 1.20 | visual escape 1.15 | spa/wellness 1.00
                                               | local/culture 0.95 | tips/guides 0.90   (keyword-matched)
           x (1 + noise)

engagement = reach
           x er_base                           reel 0.027 | carousel 0.014 | image 0.013
           x caption_multiplier                x1.05 if EN caption 80-300 chars; x0.95 if > 2200;
                                               x1.05 if both AR + EN captions present
           x (1 + noise)

split      : likes 78% | saves 10% | comments 7% | shares 5%
reels extra: views = reach x 1.1; watch_through = clamp(0.35 + 0.1 x gauss, 0.1, 0.7)
er         = engagement / reach
```

`measured_at` is a fixed constant (`sim-v1-deterministic`) so repeated runs produce identical rows - this is the reproducibility proof used in testing.

## Where the constants come from (honest ranges, cited)

- Baseline engagement rates by format reflect published cross-industry Instagram benchmarks: reels around 2.7%, static images around 1.3%, carousels around 1.4% (Sprout Social industry benchmarks), against a hospitality-sector median ER around 0.5% and a "strong" band of about 2%+ (Social Insider hospitality benchmarks; Dash Social travel benchmarks). Benchmarks vary by methodology and denominator; the simulator uses them as directional structure, not as claims about any real account.
- Evening prime-time advantage (20:00-21:30 Cairo) reflects common guidance for MENA/Egypt audiences; it is deliberately encoded as the "hidden optimum" the loop should discover.
- The pillar affinities are assumptions of the exercise (booking-intent and visual-escape content outperform tips content for a resort audience), documented here precisely because they are assumptions.

## What the uplift number does and does not prove

The before -> after optimization uplift reported in the README is computed on this simulator. It **does** prove the closed loop works end-to-end: measurement produces per-dimension differences, the Analyst extracts correct directives from them, the next batch actually applies those directives, and the resulting content scores higher on the same fixed model. It **does not** prove real Instagram performance gains - that requires the live insights path (see `go-live-path.md`), which writes the exact same Metrics schema with `source=live`.
