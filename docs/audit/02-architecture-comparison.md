# 02 — Architecture Comparison

Evidence: file trees and manifests from the commits listed in [01-feature-matrix.md](01-feature-matrix.md).

| Aspect | pendpost | SimplePost | Postiz | Open-Dispatch |
|---|---|---|---|---|
| Language/runtime | Node ≥20, ESM `.mjs`, **zero runtime npm deps in root `package.json`**; React/Vite dashboard (`app/`) | TypeScript monorepo (yarn 4): `sdk`, `server`, `scheduler`, `cli`, `e2e` | TypeScript monorepo (Nest backend, Next frontend, orchestrator) | Python 3.11, FastAPI |
| Shape | Monolith daemon + per-platform CLI scripts (`scripts/meta-social.mjs` 1723 lines, `yt-social.mjs` 1957) | **Modular**: publishing SDK is separable from the hosted-style scheduler app | Monolith-ish services, large | Small modular: `adapters/`, `scheduler/`, `api/`, `ai/` |
| Adapter pattern | Weak: platform logic in big scripts + `lib/` glue | **Strong**: `sdk/src/publishers/{instagram,facebook,youtube,…}` with `validation.ts` per platform and a `base/` class | Strong: `social.integrations.interface.ts` + one provider per platform | Strong: one adapter per platform, shared `errors.py` |
| Scheduler | In-process `lib/scheduler.mjs` (2030 lines) plus native platform scheduling (FB, YT) | Postgres-backed scheduler app; `account-lock.ts` serializes per account | Separate `apps/orchestrator` worker app (queue tech not verified; `package.json` matched "temporal|bullmq") | `scheduler/` module |
| Queue | None external; file-based state | Postgres via Prisma | UNKNOWN (not verified) | internal (not traced) |
| Database | JSON/files under `data/` | PostgreSQL (Prisma, `schema.prisma`) | PostgreSQL (Prisma) | UNKNOWN (not traced) |
| Media storage | Local disk, streamed to platform resumable endpoints | S3 SDK deps (`@aws-sdk/client-s3`), `server/src/services/uploads.ts` | UNKNOWN | Local/`media/`; IG needs public URLs |
| Auth to AutoCast itself | Loopback bind, Host/Origin checks; no multi-user | User/Session/Account tables in `schema.prisma`, API keys, CLI tokens, MCP tokens | Multi-user | API key (not traced) |
| Platform OAuth | Env tokens; PKCE for X; YT OAuth script | `scheduler/lib/oauth` (PKCE), pending connections | Yes | Static tokens |
| Token storage | **Plaintext `.env`** (0600 for some files); display shows only 4-char tail | `lib/security/encryption.ts` and `connected-account-secrets.ts` exist (algorithm not reviewed) | UNKNOWN | `.env` |
| Retry model | Error-368 circuit breaker; cadence cap | Retry only for operations strictly before `media_publish`; ambiguous post-publish failures are not blindly retried (comment in `instagram/index.ts:546`) | UNKNOWN | transient vs permanent classification |
| Event model | Notify module, receipts | `WebhookEndpoint` model | UNKNOWN | UNKNOWN |
| Plugin architecture | Rules JSON (`rules.json`) for lint | Plugin/extension points exist (`PLUGIN_EXTENSIONS` feature, agent plugins) | Provider interface | Adapter registry |
| MCP | `lib/mcp.mjs`, dashboard/MCP parity test | MCP server + skills | separate agent repo | `mcp_server.py` |
| AI integration | Agent is external; MCP is the interface. Optional internal agent-runner and radar modules | Skills for agents; core SDK has no LLM SDK in deps | Built-in AI features (not traced) | `ai/` optional |

## Patterns worth reusing (as ideas; all from MIT repos)

1. **Fail-closed approval state** (`draft/approved/rejected`, creator cannot approve) — pendpost `DISCLAIMER.md`, `SECURITY.md`. This is the closest existing match to AutoCast's "approval workflow".
2. **Per-platform `validate()` before publish** — SimplePost `validation.ts` files, pendpost `lib/lint.mjs`.
3. **Native platform scheduling where it exists** (FB `scheduled_publish_time`, YT `private + publishAt`) so posts survive server downtime — pendpost `meta-social.mjs:651-682`, `yt-social.mjs:439-441`.
4. **Retry only pre-publish; treat post-publish ambiguity as "unknown, reconcile"** to avoid duplicate posts — SimplePost IG publisher.
5. **Account-ban circuit breaker** (stop the lane on Meta error 368) — pendpost.
6. **Per-account publish lock** — SimplePost `scheduler/lib/posting/account-lock.ts`.
7. **Interface parity: every capability on UI and MCP** — pendpost parity test.

## What no candidate gives for free

A clean, small, library-shaped core that is (a) MIT, (b) covers IG Reels + FB Reels + YT resumable with thumbnails and captions, (c) has no hosted-SaaS baggage. SimplePost's SDK comes closest to the library shape but lacks verified FB Reels and YT captions; pendpost has the fuller Meta/YT API coverage but in script form.
