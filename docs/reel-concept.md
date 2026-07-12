# Full Reel Concept - Coral Garden Encounter

> The required full Reel deliverable, generated on-brand by the Content Studio (batch 1, post b1-p2),
> judged 4-5/5 by the cross-model Brand Judge and human-approved.

- **Pillar:** Aqua Adventures
- **Publish slot:** 2026-07-14T15:00:00.000+03:00 (afternoon-dreamscape)
- **Hook (first 2 seconds):** Watch as the water reveals a hidden world of color and life.
- **Audio direction:** Calm tropical house build-up with subtle underwater ambient sounds and uplifting melodic progression.
- **Cover text:** Dive Into Color

## Captions

**English:**
> Beneath the surface lies an artist's palette. Vibrant coral gardens, dancing fish, pure wonder. Your underwater masterpiece awaits—link in bio to explore.

**Egyptian Arabic (written natively):**
> تحت المية في Insta Bay بتتحول لحدائق ملونة 🐠🌊
> شعاب مرجانية، سمك ملون، وطاقة طبيعية تخبل.
> هتقول إيه لما تشوفها بنفسك؟
> DM عشان نساعدك تحجز غطسة تنسيها.

**Hashtags:** #InstaBay #Hurghada #RedSeaEgypt #Snorkeling #DivingEgypt #UnderwaterWorld #MarineLife #WaterSports #CoralReef

## Storyboard (15s, 5 scenes)

| # | Shot | Action | On-screen text | Sec |
|---|------|--------|----------------|-----|
| 1 | Wide underwater pan | Camera glides over coral reef, sunlight streams from above | Beneath the Surface | 3 |
| 2 | Close-up coral detail | Zoom in on vibrant coral formations, fish dart past | Nature's Canvas | 3 |
| 3 | Fish school movement | School of colorful fish moves across frame in coordinated motion |  | 3 |
| 4 | Snorkeler POV | Perspective shifts to snorkeler gazing at reef, hand reaches toward coral (not touching) | Your Moment Awaits | 3 |
| 5 | Wide escape | Camera pulls back to show full reef ecosystem, diver silhouetted against sunlit water | Insta Bay | 3 |

## Publishing path (already wired in the Publisher workflow)

1. Produce the 9:16 MP4 from this storyboard and host it at a public HTTPS URL.
2. `POST /{ig-user-id}/media` with `media_type=REELS`, `video_url`, caption - returns the container ID.
3. Poll `GET /{container-id}?fields=status_code` (~1/min) until `FINISHED` (early publish returns error 9007).
4. `POST /{ig-user-id}/media_publish` with `creation_id`.