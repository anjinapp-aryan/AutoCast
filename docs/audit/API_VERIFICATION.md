# Official API Verification — Phase 0.5

Date: 2026-10-27. Source: official Meta and Google documentation pages fetched this phase. Pages carry their own "last updated" stamps (Google: 2026-10-15/16). Quotes are from those pages.

## Meta (Instagram + Facebook)

Current Graph API version: **v26.0** (introduced 2026-07-29). Source: https://developers.facebook.com/docs/graph-api/changelog . Candidate code uses v24.0 (pendpost) and v25.0 (SimplePost); v24.0 and v25.0 are still listed as available (to 2028), so no urgency; pin and track.

### Instagram Content Publishing
Docs: https://developers.facebook.com/docs/instagram-platform/content-publishing , https://developers.facebook.com/documentation/instagram-platform/instagram-graph-api/reference/ig-user/media , .../ig-user/media_publish , .../ig-container , .../ig-user/content_publishing_limit

| Topic | Verified result |
|---|---|
| Flow | `POST /<IG_ID>/media` (container) → poll `GET /<CONTAINER>?fields=status_code` → `POST /<IG_ID>/media_publish {creation_id}`. Status values: EXPIRED, ERROR, FINISHED, IN_PROGRESS, PUBLISHED. Docs recommend polling once per minute for at most 5 minutes. Containers expire after 24 h. |
| Media input | URL path: "must be on a public server". Resumable path: `upload_type=resumable`, then POST bytes to `rupload.facebook.com/ig-api-upload/<VERSION>/<CONTAINER_ID>` with `offset` and `file_size` headers (local file or `file_url`). So local-disk upload is official. |
| Reels | `media_type=REELS`. MOV/MP4, H264/HEVC, 23–60 fps, max width 1920, aspect 0.01:1–10:1 (9:16 recommended), 3 s–15 min, 300 MB, AAC audio. Reels cannot be in carousels. |
| Permissions | Facebook Login (Page token): `instagram_basic`, `instagram_content_publish`, `pages_read_engagement`. Instagram Login: `instagram_business_basic`, `instagram_business_content_publish`. User needs MANAGE or CREATE_CONTENT task on linked Page. |
| Account | Professional (business/creator) account required. |
| App Review | Standard Access is default and "intended for apps that will only be used by people who have roles on them"; if the app "only serves your Instagram professional account or an account you manage, Standard Access is all your app needs". Advanced Access needs App Review + Business Verification and is required to serve accounts you do not own. The docs do not explicitly say a dev-mode app can publish; the wording implies yes for own accounts. **UNKNOWN until tested with a real account.** |
| Publishing limit | **Conflict inside Meta's own docs**: guide Rate Limit section says 100 posts per 24 h; guide carousel section, `media_publish` reference and `content_publishing_limit` reference (`quota_total` "currently 50") say 50. Treat 50 as the working number and read `GET /<IG_USER_ID>/content_publishing_limit` at runtime. Also 400 containers per rolling 24 h. |
| Idempotency | **None documented.** `media_publish` takes only `creation_id` and token. Recovery path per docs: if publish returns no media id, query container `status_code`; `PUBLISHED` means done. Docs do not say what a second `media_publish` on the same container does. **UNKNOWN.** |
| Partial success | `media_publish` can return `config_issue` (CAPTION_NOT_ATTACHED, USER_TAGGING_FAILURE) while the media is published. |

### Facebook Pages
Docs: https://developers.facebook.com/docs/video-api/guides/publishing , https://developers.facebook.com/documentation/video-api/guides/reels-publishing , https://developers.facebook.com/docs/graph-api/reference/page/videos , .../page/video_reels

| Topic | Verified result |
|---|---|
| Page video | `POST graph-video.facebook.com/v26.0/<PAGE_ID>/videos`. Reference lists upload phases `start/transfer/finish/cancel` (SimplePost and pendpost both use this). Newer guide describes a generic resumable-handle flow instead. |
| Page Reels | 3-phase `POST /<page-id>/video_reels` (`start` → upload to `rupload.facebook.com/video-upload/<version>/<video-id>` → `finish` with `video_state` DRAFT/SCHEDULED/PUBLISHED). Only to Pages. 9:16, 540×960 min, 24–60 fps, **3–90 s**. |
| Scheduling | Videos: `scheduled_publish_time` between 10 min and 6 months ahead. Reels: >10 min and within 29 days, `video_state=SCHEDULED`. |
| Permissions | `pages_show_list`, `pages_read_engagement`, `pages_manage_posts`; Page token from user with CREATE_CONTENT. `publish_video` not found on any page read (Phase 0 / older posts mention it). |
| Rate limit | Reels API: **30 API-published posts per 24 h**. |
| Tokens | User tokens: short-lived 1 h, long-lived 60 days. Page-token lifetime not confirmed from a Page-specific page: **UNKNOWN**. pendpost recommends a Business System User token to mint a non-expiring Page token (its own claim, not verified against Meta). |

## Google / YouTube
Docs: https://developers.google.com/youtube/v3/determine_quota_cost , .../docs/videos/insert , .../guides/using_resumable_upload_protocol , .../docs/thumbnails/set , .../docs/captions/insert , .../guides/quota_and_compliance_audits , https://developers.google.com/identity/protocols/oauth2

| Topic | Verified result |
|---|---|
| Quota | `videos.insert`: "100 quota per day. Each call costs 1 quota." Separate bucket. `thumbnails.set` ≈ 50 units, `captions.insert` 400, `videos.list` 1. Default: 100 `videos.insert` calls/day + 10,000 units/day for other endpoints. Resets midnight PT. |
| Upload | Resumable, `POST https://www.googleapis.com/upload/youtube/v3/videos`, max file 256 GB. Scope `youtube.upload` suffices. |
| Resume | Empty PUT with `Content-Range: bytes */SIZE` → 308 with `Range`, or the original completion response if finished. Session URI "eventually expires" (no duration given); expired → 404, start over. |
| `publishAt` | Settable via `status.publishAt`; error `invalidPublishAt` exists. The reference page does **not** state the "must be private" rule; both candidates set `private` + `publishAt`. Keep that, unverified from docs. |
| Thumbnails | `thumbnails.set`, 50 MB, JPEG/PNG. Channel-verification requirement NOT on the page: **UNKNOWN**. |
| Captions | `captions.insert` needs scope `youtube.force-ssl` (or `youtubepartner`); `youtube.upload` alone does **not** work. Required: videoId, language, name; 409 `captionExists`. |
| OAuth | External app in "Testing" status gets refresh tokens that expire in **7 days**; cap of 100 test users. Published-unverified with sensitive scopes: 100 total user cap. Production refresh token has no stated lifetime; stops working after 6 months unused, password change with Gmail scopes, or revocation. 100 refresh tokens per account per client ID. |
| Idempotency | None for `videos.insert`. Recovery only through the live session URI. |

### Claim reconciliation (Google)

```text
Claim: videos.insert quota cost
Phase 0 result: reported inconsistently (1600 vs 100 vs 1)
Official documentation: determine_quota_cost + videos/insert
Verified result: 1 unit, in its own bucket capped at 100 uploads/day by default (no 1600 on current page)
Confidence: HIGH (two official pages agree)

Claim: unverified API projects force uploads private
Phase 0 result: "uploads from unverified API projects are locked private until a compliance audit" (Ayrshare blog, secondary)
Official documentation: videos/insert page says "Videos uploaded from unverified API projects are not restricted to private viewing mode."
Verified result: official page contradicts Phase 0. Real-world behavior may differ from the page (the audit guide URL used in Phase 0 returned 404, and the older July-2020 restriction text was not found anywhere).
Confidence: MEDIUM for the docs claim; real behavior UNKNOWN until a test upload from a fresh project.

Claim: Instagram publishing limit
Phase 0 result: 100 vs 50
Official documentation: Meta pages conflict (100 in one section; 50 in three others)
Verified result: use 50 and read content_publishing_limit at runtime
Confidence: MEDIUM
```

## Official-docs summary

Both official paths are viable for a single-user self-hosted tool publishing to accounts the user owns. What stays UNKNOWN until a live spike: dev-mode IG/FB publishing, YouTube unverified-project visibility, Page-token lifetime.
