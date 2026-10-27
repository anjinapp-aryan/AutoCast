# 06 — Build vs Reuse

| Component | Decision | Candidate | Reason |
|---|---|---|---|
| Instagram publishing | COMPOSE | `@simple-post/sdk` + pendpost resumable path | See [03](03-reuse-decisions.md). Need to confirm SDK handles local-file upload; otherwise port pendpost's `rupload.facebook.com` flow. |
| Facebook publishing | REUSE (video/post), ADAPT (Reels) | SDK; pendpost `video_reels` | Only pendpost has verified Reels flow. |
| YouTube publishing | REUSE | SDK + `googleapis` | Resumable, thumbnails, publishAt verified in source. |
| YouTube captions | ADAPT | pendpost `yt-social.mjs` | ~25 lines; SDK lacks it. |
| Scheduler | COMPOSE | `pg-boss` / `bullmq` | General-purpose, MIT. Spike required to choose. |
| OAuth | COMPOSE | `googleapis` OAuth2; Meta via plain HTTP | No third-party OAuth stack required. |
| Media handling | COMPOSE | `sharp` (Apache-2.0) for image checks; `ffprobe` (external binary, LGPL/GPL by build — **license review needed**) for video validation | Video probing library choice is an open question. |
| Queue | COMPOSE | same as Scheduler | |
| Database | COMPOSE | PostgreSQL (or SQLite for mock MVP) + an ORM | Commodity. |
| Token vault | BUILD (thin) | — | Security-critical, small; use platform crypto, no copy of candidate code. |
| Approval workflow | BUILD | rules from pendpost (REFERENCE) | AutoCast-specific domain; pendpost's implementation is tied to file layout. |
| Campaign model | BUILD | — | AutoCast-specific. |
| Audit log | BUILD | — | Tiny, tied to schema. |
| Circuit breaker / rate limits | ADAPT | pendpost 368 breaker, SDK account lock | Small, valuable. |
| AI captions | COMPOSE (optional) | provider SDKs behind interface | Not needed for core. |
| Analytics | ADAPT (later) | pendpost `insights.mjs` as reference | Phase 11. |
| MCP | COMPOSE | `@modelcontextprotocol/sdk` | AutoCast writes tool definitions only. |
| REST API/CLI | BUILD (thin) | SDK `server/`, `cli/` as REFERENCE | Avoid inheriting their auth/billing. |
| Dashboard | BUILD on UI kit | — | Both candidate UIs are bound to their backends. |

## BUILD justifications

| Item | Why not reuse |
|---|---|
| Token vault | No candidate offers an isolated, reviewed vault; pendpost is plaintext, SimplePost's is embedded in a Prisma SaaS app and its algorithm was not reviewed. Security-critical code should be small and owned. |
| Approval workflow, campaigns, audit log | Genuinely AutoCast-specific and tied to AutoCast's data model; closest implementation (pendpost) is MIT but file-store-bound. Rules are reused as REFERENCE. |
| REST/CLI/Dashboard | Existing ones would import another product's identity, billing and storage model. |
| Shorts handling | Trivial; no code worth copying. |
