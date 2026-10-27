# 03 — Reuse Decisions

Classes: REUSE / ADAPT / COMPOSE / REFERENCE / BUILD / REJECT. One per capability. Evidence column points to files inspected (see [01-feature-matrix.md](01-feature-matrix.md) for commits).

| Capability | Decision | Source | Evidence / reason |
|---|---|---|---|
| Instagram publishing (image/Reels, container → poll → publish) | **COMPOSE** | `@simple-post/sdk` (MIT) as primary; pendpost `meta-social.mjs` (MIT) for the resumable upload path | SDK: `instagram/index.ts` implements container status polling, `media_publish`, REELS, and a careful pre/post-publish retry split. pendpost adds `upload_type: resumable` against `rupload.facebook.com` from local disk, so no public URL is required. SDK's path for local files was not verified. |
| Facebook Page video / photo / text | **REUSE** | `@simple-post/sdk` | `facebook/video-upload.ts` does start/transfer/finish, supports `scheduled_publish_time`. |
| Facebook Reels (`/video_reels`) | **ADAPT** | pendpost `meta-social.mjs:198-243` | Only pendpost has the verified 3-phase `video_reels` flow. It is embedded in a 1,700-line script and must be extracted. |
| YouTube upload (resumable, publishAt, thumbnail) | **REUSE** | `@simple-post/sdk` YouTube publisher; `googleapis` (Apache-2.0) | `youtube/resumable-upload.ts` + `thumbnails.set` + `publishAt` all present. |
| YouTube captions (SRT) | **ADAPT** | pendpost `yt-social.mjs:387-410` | SDK has no `captions` hit. About 25 lines of logic; or call `captions.insert` through `googleapis`. |
| YouTube Shorts | **BUILD (trivial)** | — | A Short is a vertical ≤60s upload; no candidate has special logic except Open-Dispatch. Only validation/metadata (`#Shorts`) is AutoCast-specific. |
| Platform validation (length, ratio, size) | **REUSE** | SDK `validation.ts` per platform | Present for IG, FB, YT. |
| OAuth flows (Google) | **COMPOSE** | `googleapis` OAuth2 client | Official library; no need to copy anyone's flow. |
| OAuth / token exchange (Meta long-lived tokens) | **ADAPT** | SDK `scheduler/lib/oauth` (reference) + Meta docs | Not verified end-to-end in any candidate. |
| Token storage encryption | **BUILD (thin) on a standard lib** | Node `crypto` AES-GCM / libsodium | pendpost stores plaintext `.env`; SimplePost has `lib/security/encryption.ts` but is embedded in a Prisma app. Algorithm not reviewed. Do not copy blindly. |
| Scheduler / job queue | **COMPOSE** | `pg-boss` (MIT, v12.38.0) or `bullmq` (MIT, v6.5.0) | Neither was source-audited here; both are mature general libraries and avoid writing a scheduler. Choice deferred to Phase 1 with a spike. |
| Native-schedule strategy (FB, YT) | **REFERENCE** | pendpost | Idea only: use platform scheduling where it exists. |
| Per-account lock, retry classification | **REFERENCE** | SimplePost `account-lock.ts`, IG retry predicates | Small designs; reimplement. |
| Approval workflow (draft/approved/rejected, no self-approval) | **REFERENCE → BUILD** | pendpost `DISCLAIMER.md`/`SECURITY.md`, `lib/state.mjs` | pendpost's gate is welded to its file store and plan format; AutoCast's domain model is different. Build on the same rules. |
| Campaign domain model | **BUILD** | — | AutoCast-specific. No candidate has a reusable standalone model (pendpost has plans, tied to its file layout). |
| Audit log | **BUILD** (append-only table) | pendpost `attest.mjs` as REFERENCE | Small and tied to AutoCast schema. |
| Anti-ban circuit breaker (Meta 368) | **ADAPT** | pendpost | Small, high-value rule; port the logic. |
| REST API | **BUILD (thin)** | SimplePost `server/` is REFERENCE | Reusing the server drags its auth/billing model. |
| MCP server | **COMPOSE** | `@modelcontextprotocol/sdk` (MIT, v1.32.2) | Official SDK; AutoCast writes only the tool definitions. |
| CLI | **BUILD (thin)** | — | Wrapper over the REST API. |
| Dashboard UI | **BUILD**, with component libraries | — | pendpost dashboard (React/Vite/Radix) and SimplePost scheduler (Next.js) are tied to their own data models. Reuse UI kits, not their apps. |
| AI captions/metadata | **COMPOSE** | Provider SDKs, optional | Must be an optional plug-in. See [04-ai-dependency.md](04-ai-dependency.md). |
| Analytics | **ADAPT** later | pendpost `insights.mjs` as REFERENCE | Platform insights endpoints; not Phase 1. |
| Whole-product reuse (run Postiz/Mixpost/SimplePost scheduler as AutoCast) | **REJECT** as a base | Postiz AGPL; Mixpost Lite lacks YouTube and is stale; SimplePost scheduler is a hosted SaaS app (Stripe billing models in schema). | |
| Postiz code | **REJECT** (direct) / **REFERENCE** architecture only | AGPL-3.0 `LICENSE` | RED. |
| Mixpost Lite code | **REFERENCE** | MIT but PHP/Laravel; no YouTube; last commit 2026-03-16; "Lite" of a commercial product, README says Lite features are kept distinct from Pro | |
| Open-Dispatch | **REFERENCE** | MIT; IG adapter is 130 lines, needs public media URLs, Graph v21; YouTube Shorts only | Too thin to depend on. |
| RolePlayingTech/social-media-manager | **REJECT** | **No LICENSE file** (GitHub `license: null`) | Cannot be reused. Also uploads media to a third-party host (`tmpfiles.org`, `publisher.py:37-45`) and stores page/refresh tokens in SQLite columns (`database.py:46,61`). |
| Browser/cookie automation (pendpost `engage-browser`, `easy-uploader` Selenium scripts, growchief) | **REJECT** for publishing | See [FINAL-REPORT.md](FINAL-REPORT.md) §Platform API | ToS and ban risk; not needed because official APIs cover IG, FB, YT publishing. |
| `facebook-nodejs-business-sdk` | **REJECT** | npm license field: "Platform License" | Not OSI; do not depend. Call Graph API over HTTP. |
