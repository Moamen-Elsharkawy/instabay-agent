# Go-Live Path: from dry-run to real Instagram publishing

The system ships in `dry_run` mode: the Publisher workflow logs the exact Graph API request sequence it would send (per post, in the Calendar's `api_payload_log` column) instead of calling Meta. The live branch is fully built and parameterized in the same workflow - going live is a config flip, not a rebuild.

## Why this is a safe claim

Everything the live branch needs is already wired: quota pre-check, media-type routing (reel vs image vs carousel), the 3-step Reels state machine (create container, poll `status_code`, publish), and result normalization back into the Calendar. The only missing inputs are real credentials and a real Instagram professional account.

## Step-by-step (about 30-45 minutes)

1. **Instagram professional account.** Convert the resort's Instagram account to a Business account (Settings, Account type). Since the July 2024 "Instagram API with Instagram Login" product, **no linked Facebook Page is required** for content publishing and insights.
2. **Meta app.** Create an app at developers.facebook.com, add the "Instagram" product, and select **Instagram API with Instagram Login**.
3. **Permissions/scopes.** The app needs `instagram_business_basic` and `instagram_business_content_publish` (add `instagram_business_manage_comments` later for comment automation). Note: the older scope names (`instagram_basic`, `instagram_content_publish`) were deprecated for this product in January 2025.
4. **Development mode is enough for a pilot.** Accounts with a role on the app (admin/developer/tester) can publish through the API **without App Review**. Full App Review is only needed to serve accounts that do not have a role on the app (i.e., when this becomes a multi-client product).
5. **Token.** Generate a long-lived access token for the resort's account (60-day validity; refresh it via the token refresh endpoint - schedule an n8n cron for this before day 50).
6. **Config flip.** In the Command Center sheet's `Config` tab set:
   - `ig_user_id` = the Instagram professional account ID
   - `ig_access_token` = the long-lived token (for production, move this into an n8n credential or environment variable instead of the sheet - the sheet placement is a demo convenience and is called out as such)
   - `publish_mode` = `live`
7. **Media hosting.** Meta fetches `image_url` / `video_url` over public HTTPS. The pipeline already generates public HTTPS image URLs (Pollinations). For production-grade visuals, swap in the resort's own photography uploaded to any public host (Cloudinary free tier, S3, or Drive with public links). Reels need a hosted MP4 (9:16, up to ~90s): produce it from the generated storyboard, host it, and put the URL in a `video_url` column.
8. **First live test.** Approve a single post, run the Publisher with `{"force": true}`, and verify the post appears on the account and that `media_id` + `published_at` land in the Calendar.

## Rate limits (already enforced)

- Instagram allows **100 API-published posts per rolling 24 hours** per account. The live branch checks `GET /{ig-user-id}/content_publishing_limit` before publishing and aborts if the quota is exhausted.
- A carousel counts as one post against the quota.
- Containers expire after 24 hours if not published; reel containers must reach `status_code = FINISHED` before `media_publish` (publishing early returns error 9007). The Publisher polls once per minute with a 10-poll guard.

## Insights (measurement goes live the same way)

The Analyst workflow's live branch calls `GET /{ig-media-id}/insights?metric=reach,likes,comments,saved,shares,views` per published post and writes the same Metrics schema the simulator fills today (`source=live` instead of `source=synthetic`). Account-level metrics (`follower_count`, `reach`, `accounts_engaged`) can be added from `GET /{ig-user-id}/insights` to replace the static `follower_base` config.

## Known gaps in the live branch (honest list)

- **Carousel live publishing** is stubbed (needs a child-container loop: one container per slide with `is_carousel_item=true`, then a `CAROUSEL` container with `children`, then publish). The dry-run logs this exact sequence; implementing it is ~1 hour.
- The live branch is **validated but not executed** (no real Business account existed inside the challenge time-box). The dry-run path and every other workflow ran end-to-end for real.
- Token storage should move from the Config sheet to an n8n credential before any production use.

## Note on "Meta MCP"

As of mid-2026, Meta's official MCP server (Meta Ads AI Connectors, open beta April 2026) covers the **paid Marketing/Ads API only** - it does not publish organic feed posts or reels. Organic publishing still goes through the Graph API directly, which is what this system integrates. The pipeline itself could be exposed as an MCP server later so any agent (Claude, etc.) can drive it.
