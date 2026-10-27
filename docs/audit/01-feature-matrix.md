# 01 — Feature Matrix

Legend: YES = seen in source code; PARTIAL = some support seen or constrained; NO = searched, not found; UNKNOWN = not verified. README claims alone were never counted as YES.

Candidates inspected from shallow clones (depth 1) in the session scratchpad:

| Short name | Repo | Commit inspected |
|---|---|---|
| **pendpost** | pendpost/pendpost (v2.7.0) | 42ac77b, 2026-10-14 |
| **SimplePost** | simple-post/core (`@simple-post/sdk` 1.5.0) | 957ed6a, 2026-10-27 |
| **Postiz** | gitroomhq/postiz-app (v2.25.3) | a5affe6, 2026-10-24 |
| **Mixpost Lite** | inovector/mixpost (2.6.0) | df57648, 2026-03-16 |
| **Open-Dispatch** | Matthew-Selvam/Open-Dispatch | 773dd54, 2026-10-18 |
| **RP-SMM** | RolePlayingTech/social-media-manager (the IG+FB+YT "Social Media Manager"; other repos of that name exist) | f0c2801, 2026-10-26 |

| Capability | pendpost | SimplePost | Postiz | Mixpost Lite | Open-Dispatch | RP-SMM |
|---|---|---|---|---|---|---|
| Instagram | YES (`scripts/meta-social.mjs`) | YES (`sdk/src/publishers/instagram`) | YES (`instagram.provider.ts`) | PARTIAL (Meta provider; not traced) | YES (`adapters/instagram.py`) | YES (`publisher.py`) |
| Instagram Reels | YES (`media_type: REELS`, `upload_type: resumable`) | YES (`REELS`; VIDEO for carousel children) | PARTIAL (REELS string present) | UNKNOWN | YES (`REELS`, needs public URL) | YES (`REELS`) |
| Facebook | YES | YES | YES | PARTIAL | YES (`facebook.py`) | YES |
| Facebook Video | YES (multipart to `graph-video`) | YES (`facebook/video-upload.ts`, start/transfer/finish) | UNKNOWN | UNKNOWN | PARTIAL (README: "text, photo, video") | UNKNOWN |
| Facebook Reels | YES (`POST /{page}/video_reels`, 3-phase) | UNKNOWN (video upload yes; `video_reels` endpoint not seen) | NO (0 hits in `facebook.provider.ts`) | UNKNOWN | NO | UNKNOWN |
| YouTube | YES (`scripts/yt-social.mjs`) | YES (googleapis + own resumable) | YES | NO (no "youtube" string in `src/`) | YES (Shorts only) | YES (OAuth2) |
| YouTube Shorts | PARTIAL (no Shorts-specific logic seen; Shorts are normal uploads of vertical ≤60s video) | PARTIAL (same) | UNKNOWN | NO | YES (by design) | UNKNOWN |
| Scheduling | YES (native FB `scheduled_publish_time`, YT `publishAt`; IG/LI/X need process running) | YES (DB + scheduler app) | YES | YES | YES (`scheduler/`) | YES (APScheduler) |
| Immediate publishing | YES | YES | YES | YES | YES | YES |
| OAuth | YES (YT, X PKCE; Meta via tokens) | YES (`scheduler/lib/oauth`, PKCE) | YES | YES | PARTIAL (static tokens; LinkedIn 3-legged) | PARTIAL (YT OAuth2 built in) |
| Token refresh | YES (X, YT refresh) | YES (`credential-health.ts`) | UNKNOWN | UNKNOWN | UNKNOWN | PARTIAL (YT refresh token stored) |
| Multi-account | YES (`lib/multi-client.mjs`) | YES (`ConnectedAccount`) | YES | YES | UNKNOWN | YES |
| Media upload | YES (local disk, resumable) | YES (S3 deps, `uploads.ts`) | YES | YES | PARTIAL (IG needs public URLs) | PARTIAL (uploads to third-party host tmpfiles.org, see 03/FINAL) |
| Large files | YES (resumable) | YES (YT resumable, FB chunked) | UNKNOWN | UNKNOWN | PARTIAL (YT resumable) | UNKNOWN |
| Thumbnail | YES (YT `thumbnails.set`, FB `/thumbnails`) | YES (YT `thumbnails.set`) | UNKNOWN | UNKNOWN | NO | UNKNOWN |
| Subtitles | YES (YT `captions.insert` SRT) | NO (no `captions` hit in YT publisher) | UNKNOWN | NO | NO | NO |
| Captions (text) | YES | YES | YES | YES | YES | YES (separate IG/FB captions) |
| Hashtags | YES (config + lint) | PARTIAL (in text; YT tags yes) | YES | YES | UNKNOWN | UNKNOWN |
| Retry | YES | YES (retry predicates pre-`media_publish`; unknown-failure after publish not retried) | YES | YES (job fail traits) | YES (`errors.py` transient/permanent) | UNKNOWN |
| Rate limiting | YES (cadence cap, 368 breaker) | PARTIAL (`account-lock.ts`) | UNKNOWN | YES (`HasSocialProviderJobRateLimit`) | UNKNOWN | PARTIAL (daily limits) |
| Queue | PARTIAL (in-process scheduler, file state) | YES (DB-backed) | PARTIAL (`apps/orchestrator` worker app; queue tech not verified) | PARTIAL (Laravel job classes seen) | YES | YES (SQLite queue) |
| Approval | YES (fail-closed gate, no self-approval) | NO (not found) | UNKNOWN | PARTIAL (Pro feature per README split; UNKNOWN) | NO | NO |
| Campaigns | YES (plans/campaigns) | NO | PARTIAL | NO | NO | NO |
| Analytics | YES (`lib/insights.mjs`) | PARTIAL (product analytics; post insights UNKNOWN) | YES | YES | NO | NO |
| Audit logs | YES (`attest.mjs`, receipts) | UNKNOWN | UNKNOWN | UNKNOWN | UNKNOWN | NO |
| Docker | YES | YES | YES | UNKNOWN (Laravel pkg; community image) | YES | NO (systemd example) |
| REST API | YES (`lib/api.mjs`) | YES (`server/`, OpenAPI) | YES | YES | YES | YES (`api.py`) |
| CLI | YES | YES (oclif) | YES (`apps/commands`, agent) | NO | YES (`cli.py`) | NO |
| MCP | YES (`lib/mcp.mjs`) | YES (`mcp.json`, server) | PARTIAL (separate `postiz-agent`) | NO | YES (`mcp_server.py`) | NO |
| Webhooks | PARTIAL (notify) | YES (`WebhookEndpoint`) | UNKNOWN | UNKNOWN | UNKNOWN | NO |
| Tests | YES (388 files in `test/`) | YES (257 test/spec files) | UNKNOWN (0 matched by my glob; jest configured) | YES (24) | YES (`tests/`; "129 tests" is README-only) | PARTIAL (1 file) |

## AutoCast column

AutoCast currently has **no code**, so every row is NO today. Target for the rows is in [06-build-vs-reuse.md](06-build-vs-reuse.md).

## Caveats

- Test counts are file counts from a glob, not coverage or pass rates. Tests were not executed.
- "UNKNOWN" for Postiz often means I did not read the file deeply, not that the feature is absent. Postiz is AGPL (RED) so deep review was low priority.
