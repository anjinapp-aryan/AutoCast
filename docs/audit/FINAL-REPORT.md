# AutoCast Phase 0 — Final Report

Date: 2026-10-27. Scope: research and audit only. No application code, no dependencies, no commit. Supporting files: [00](00-current-repository.md) · [01](01-feature-matrix.md) · [02](02-architecture-comparison.md) · [03](03-reuse-decisions.md) · [04](04-ai-dependency.md) · [05](05-proposed-architecture.md) · [06](06-build-vs-reuse.md) · [07](07-license-decision.md).

## Executive Summary

The AutoCast repo is empty (no commits, branches, or license), so nothing constrains the design. Two MIT-licensed projects cover most of the publishing problem:

- **SimplePost** (`simple-post/core`, npm `@simple-post/sdk`): a modular TypeScript SDK with IG, FB and YouTube publishers, per-platform validation, YouTube resumable upload with thumbnails and `publishAt`, and a careful "don't retry after `media_publish`" design.
- **pendpost**: MIT, zero runtime npm deps, the only candidate with verified IG resumable Reels upload from local disk, FB `video_reels`, YT thumbnails and captions, an approval gate and a Meta action-block circuit breaker. Shaped as an app plus CLI scripts, not a library.

Answer to the final question: for the platform publishing layer, queue, MCP and OAuth, **most of the code can be reused or composed from MIT/Apache-2.0 sources**. What AutoCast must write itself is the orchestration domain: campaigns, approval workflow, publish state machine and reconciliation, token vault, audit log, thin API/CLI/UI, and the adapters. Rough size cannot be measured here; the claim is qualitative.

Caveats: candidates were read from shallow clones, tests were not run, and transitive dependency licenses are UNKNOWN.

## Existing Open-Source Landscape

Searched with `gh search repos` (many query variants) plus web search. Relevant findings:

| Project | Stars | Licence | Last push | Note |
|---|---|---|---|---|
| Postiz (`gitroomhq/postiz-app`) | 38.6k | AGPL-3.0 | 2026-10-24 | Largest, most active, 70 contributors; 40+ providers. RED. |
| Mixpost Lite | 3.9k | MIT | 2026-03-16 | PHP/Laravel; no YouTube string in `src/`; no commits in the last 90 days. |
| TryPost | 755 | AGPL-3.0 | 2026-10-27 | RED. |
| Growchief | 3.5k | AGPL-3.0 | 2025-10-13 | Browser automation. RED. |
| cogsend | 241 | MIT | 2026-10-22 | Not audited in depth. |
| SimplePost | 6 | MIT | 2026-10-26 | Young repo; npm `@simple-post/sdk` 1.5.0. |
| pendpost | 14 | MIT | 2026-10-14 | v2.7.0; created 2026-06-27. |
| Open-Dispatch (`Matthew-Selvam`) | 18 | MIT | 2026-10-18 | Small. |
| RolePlayingTech/social-media-manager | 17 | **none** | 2026-10-26 | IG+FB+YT; unusable. |
| easy-uploader, chronex, social-queue, mehmetkirkoca/social-media-manager | — | MIT/ISC | stale | Not suitable. |

**"PostBox"**: no matching project found by `gh search` or web search. Marked NOT FOUND; ask for a URL if a specific repo was intended. Other repos named "Social Media Manager" exist (e.g. `mehmetkirkoca`, a feed aggregator; and the IG/FB/YT one above); I audited both.

## Top Candidates (health scores 0–5, my judgment from inspected evidence)

| Dimension | pendpost | SimplePost | Postiz | Mixpost Lite | Open-Dispatch |
|---|---|---|---|---|---|
| Activity | 3 (12 commits/90d) | 5 (100+/90d) | 5 | 1 (0/90d) | 3 |
| Documentation | 4 | 4 | 4 | 3 | 3 |
| Tests (file presence only) | 4 (388 files) | 4 (257) | 2 (unverified) | 3 (24) | 3 |
| Issue responsiveness | 3 (1 open) | 3 (16 open, 0 stars) | 3 (222 open) | 2 | UNKNOWN→2 |
| Releases | 4 (15) | 3 (15, per-package) | 5 (100+) | 3 | 2 |
| Community | 1 (2 contributors) | 1 (2) | 5 (70) | 2 (4) | 1 (2) |
| Architecture | 3 | 4 | 3 | 3 | 3 |
| Security | 3 | 3 | 2 (a recent commit "fix: jwt bypass") | UNKNOWN→2 | 2 |
| API compatibility | 5 (Graph v24) | 3 (not verified) | UNKNOWN→3 | 2 | 2 (Graph v21) |
| **Overall (avg)** | **3.3** | **3.3** | 3.6 (but RED) | 2.3 | 2.3 |

Both MIT leaders have **tiny communities (bus factor ≈ 2)** — the main reuse risk. This argues for depending on them at the edges (adapters) and vendoring/porting small, well-understood parts with attribution rather than building AutoCast on top of them.

## License Findings

See [07](07-license-decision.md). GREEN: pendpost, SimplePost, Mixpost Lite, Open-Dispatch, cogsend. RED: **Postiz, TryPost, Growchief (AGPL)** and **RolePlayingTech/social-media-manager (no license)**. `facebook-nodejs-business-sdk` ships under a "Platform License" (RED); call the Graph API directly.

## Dependency License Findings

Direct dependencies of `@simple-post/sdk` were checked (all MIT/Apache-2.0). **Transitive licenses are UNKNOWN**: no scanner was run. `sharp`/libvips (LGPL binaries) and `ffmpeg` build flavors need review. Required follow-up in Phase 1: license scanning in CI.

## Feature Matrix

See [01](01-feature-matrix.md). Source-verified highlights:

- IG Reels: pendpost `upload_type: resumable` (`meta-social.mjs:1065-1113`); SimplePost `REELS` + status polling (`instagram/index.ts:339-447`); Open-Dispatch `REELS` (`adapters/instagram.py:93`).
- FB Reels: only pendpost (`meta-social.mjs:198-243`).
- YouTube: resumable upload + `publishAt` + `thumbnails.set` in both; `captions.insert` only in pendpost (`yt-social.mjs:387-410`).

## Architecture Comparison

See [02](02-architecture-comparison.md). SimplePost has the best adapter/validation separation; pendpost has the best safety model (approval, breaker, loopback hardening).

## Security Findings

| Candidate | Finding | Evidence |
|---|---|---|
| pendpost | Loopback bind, Host/Origin checks against DNS rebinding, secrets in `.env` (plaintext, gitignored), secret fields display-only, 0600 on some files, nothing phones home (self-declared). Single-user by design; no multi-tenant isolation. | `SECURITY.md`, `lib/config.mjs:4-14`, `lib/attest.mjs:91` |
| SimplePost | Encryption helpers for connected-account secrets exist (algorithm not reviewed); SSRF guard via `ipaddr.js` in media/card fetch; hosted-SaaS surface (billing, MCP tokens, API keys) is large. | `scheduler/lib/security/*`, `sdk/src/utils/media.ts` |
| RP-SMM | Uploads user media to public third-party host `tmpfiles.org`; tokens in SQLite columns. | `publisher.py:37-45`, `database.py:46,61` |
| Postiz | Recent security-relevant commit message "fix: jwt bypass" (`a5affe6`). Not reviewed further. | git log |
| All | Not checked: CSRF details, upload path traversal, command injection, log leakage, dependency vulnerability scans (no `npm audit` run). **UNKNOWN**. | |

No candidate's security code should be copied without review.

## Platform API Findings

- **Official APIs are sufficient** for IG (Graph API container flow), FB (Page video/Reels), YT (Data API v3 resumable). All reusable code found uses them.
- **Browser automation** (Playwright/Selenium/cookies): pendpost uses Playwright MCP for engagement lanes (`lib/engage-browser.mjs`), not publishing; easy-uploader and Growchief are browser-driven. Classification: separate, risky category. Technically brittle (UI changes, CAPTCHA), ToS and account-ban risk. **Rejected for AutoCast publishing.**
- **Constraints to plan around** (external sources, partly conflicting, verify in Phase 4/6):
  - Instagram API publishing limit: Meta's Content Publishing page says 100 posts/24 h; the `content_publishing_limit` reference says 50. Use `GET /<IG_ID>/content_publishing_limit`. [Meta docs](https://developers.facebook.com/documentation/instagram-platform/content-publishing), [reference](https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/ig-user/content_publishing_limit).
  - YouTube: uploads from unverified API projects are locked private until a compliance audit. Daily quota is per project; cost of `videos.insert` is reported inconsistently (1600 vs 100 vs 1), so check the Cloud Console. Secondary sources: [Ayrshare audit note](https://www.ayrshare.com/solutions/google-api-error-403-unverified-app-how-to-fix-the-audit-pipeline/), [quota note](https://www.ayrshare.com/solutions/how-to-bypass-the-youtube-10k-unit-limit-fixing-the-403-quota-exceeded-error.md). I did not read Google's own docs.
  - Meta App Review for `instagram_content_publish` not researched. Open item.

## LLM Dependency

See [04](04-ai-dependency.md). None of the MIT publishing paths needs an LLM. AI is an optional drafting plug-in.

## Hardware Requirements

Not benchmarked; reasoned from stack, treat as estimates.

| Tier | Estimate |
|---|---|
| Local dev | Any laptop, 8 GB RAM, Docker, internet. **No GPU.** |
| Small VPS | 1–2 vCPU, 2 GB RAM, 20+ GB disk for media; outbound bandwidth sized to video uploads (a 200 MB video × N posts/day). |
| Production | App + worker, PostgreSQL, S3-compatible storage, monitoring. SimplePost's scheduler (Next.js + Prisma + Postgres) is heavier than pendpost (no deps). |
| GPU | Only for local AI/video generation. Publishing APIs never need it. Video transcoding (ffmpeg) is CPU. |

## REUSE / ADAPT / COMPOSE / REFERENCE / BUILD / REJECT Decisions

Full table in [03](03-reuse-decisions.md) and [06](06-build-vs-reuse.md). Summary:

- REUSE: FB posting/video, YouTube upload, platform validation.
- ADAPT: FB Reels, YouTube captions, Meta 368 breaker.
- COMPOSE: IG publishing, scheduler/queue, OAuth (googleapis), MCP, media checks, AI.
- REFERENCE: approval rules, per-account lock, retry classification, native scheduling.
- BUILD: campaigns, approval workflow, audit log, token vault, thin API/CLI/UI.
- REJECT: Postiz/TryPost/Growchief code, RP-SMM, browser automation, `facebook-nodejs-business-sdk`.

## Recommended Architecture

**Model D implemented through B**: AutoCast builds the orchestration layer and a `PublisherAdapter` abstraction; platform calls go through `@simple-post/sdk` and `googleapis`, with small ported pieces from pendpost for gaps. Model A (wrap an existing app) rejected: pendpost is single-user/file-based; SimplePost's scheduler is a billing-coupled SaaS; Postiz is AGPL. Model E rejected: duplicates mature MIT code. Diagram in [05](05-proposed-architecture.md).

## Recommended Technology Stack

TypeScript/Node (matches both MIT leaders so code can be ported with minimal friction), PostgreSQL (SQLite for mock MVP), `pg-boss` or `bullmq` (spike to choose), `googleapis`, plain HTTP for Meta Graph, `@modelcontextprotocol/sdk`, Docker Compose. These are proposals; the queue choice is untested.

## Implementation Roadmap (revised)

0. Reuse audit (this).
1. Architecture + licensing: pick AutoCast license; transitive license scan in CI; queue spike; confirm `@simple-post/sdk` can publish local files and works with Reels; define `PublisherAdapter`.
2. Mock MVP: domain model, approval gate, state machine, mock adapters, audit log.
3. Asset management (local store, validation, ffprobe decision).
4. Instagram adapter (compose SDK + resumable path); Meta app setup and review.
5. Facebook adapter incl. Reels (port pendpost).
6. YouTube adapter incl. thumbnails/captions; API audit for public uploads.
7. Scheduler hardening, native schedule + reconciliation.
8. Reliability (retry classification, breaker, rate limits) — **moved earlier than the original plan's Phase 10**, since duplicate-post risk exists from the first live adapter.
9. MCP + CLI.
10. Optional AI layer.
11. Analytics.
12. Production hardening.

## Risks

1. Both MIT leaders are young with ~2 contributors; abandonment risk.
2. Platform review gates (Meta App Review, YouTube audit) can block real publishing regardless of code.
3. API limits and quota facts conflict across sources.
4. Duplicate posts when a publish call's outcome is ambiguous.
5. Unreviewed transitive licenses and security of reused code.
6. pendpost README says the repo is a subset of a commercial product; future license/feature divergence possible.

## Open Questions

1. Which license does AutoCast want (MIT/Apache-2.0 recommended)?
2. "PostBox": which repository was meant?
3. Single-user self-hosted, or multi-tenant? This changes token vault and auth scope.
4. Is IG/FB publishing from a Page-linked Business/Creator account acceptable (API requirement)?
5. Should AutoCast depend on `@simple-post/sdk` from npm, or vendor a pinned copy?

## Things We MUST NOT Build Ourselves

OAuth client libraries, Google API clients, YouTube resumable upload protocol handling (use `googleapis` / SDK), a general job queue, MCP protocol plumbing, platform field validation tables already in the SDK, image processing.

## Things We MUST Build Ourselves

Campaign/content domain model, approval workflow, publish state machine with reconciliation, adapter interface and mock adapters, token vault and key management, audit log, thin API/CLI/dashboard, per-platform gaps (FB Reels, YT captions) if not extracted from pendpost, optional AI plug-in contract.

## Method and limitations

- Evidence: shallow clones of 11 repos; `gh api` metadata; `npm view` licenses; 3 web searches. File paths cited are from those clones.
- Not done: running tests, `npm audit`, lockfile license scan, reading Meta/Google official docs directly, deep review of Postiz/Mixpost/cogsend/chronex, verifying SimplePost's encryption algorithm, benchmarking hardware.
- Scores are subjective.
