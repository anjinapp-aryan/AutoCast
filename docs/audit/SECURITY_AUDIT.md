# Security Audit — Phase 0.5

Date: 2026-10-27. Scope: focused source review of `simple-post/core@957ed6a` (SDK in depth; server/scheduler only by grep) and `pendpost@42ac77b`. This is a targeted read, **not** a penetration test. Anything not listed was not checked and is UNKNOWN.

## Checked vs not checked

| Area | SimplePost | pendpost |
|---|---|---|
| Token storage | Not reviewed in depth. SDK `utils/crypto.ts` is only an ES256 JWT helper (DER→raw), not secret storage. Scheduler encryption (`lib/security/*`) algorithm: **UNKNOWN**. | Reviewed: plaintext `.env` per client. |
| SSRF / URL validation | Reviewed (good). | Not applicable (no remote-URL fetch path inspected). **UNKNOWN.** |
| Path traversal | Not reviewed. | Reviewed (static + media). |
| Command execution | Not reviewed. | Reviewed (grep). |
| Loopback / Host / Origin | n/a (SDK library). Server: UNKNOWN. | Reviewed. |
| CSRF, rate limiting, webhooks, logging leaks | UNKNOWN | UNKNOWN (CORS/Host checks only) |
| Dependency vulns | 7 high in published 1.5.0 | none (zero deps) |

## Findings

| Severity | Finding | Evidence | Exploitability | AutoCast impact | Recommendation |
|---|---|---|---|---|---|
| High | Published `@simple-post/sdk@1.5.0` ships vulnerable `sharp`/libvips (4 CVEs) and `zod` (memory exhaustion) | `npm audit` on installed tree; [sbom/candidate-stack-npm-audit.json](sbom/candidate-stack-npm-audit.json) | libvips CVEs trigger on parsing hostile images; AutoCast would process user-provided media only. zod issue needs hostile input to validation. Low in a single-user setup, real if media comes from URLs. | Blocks "just `npm i` it" unless fixed. `main` already declares newer ranges (`sharp ^0.35.5`, `zod ^4.4.3`). | Condition: depend on a release that includes those bumps, or use `overrides`, and gate on `npm audit` in CI. |
| Medium | pendpost keeps platform secrets in a plaintext `.env` | `lib/config.mjs:4-14`, SECURITY.md ("Secrets stay in your .env") | Anyone/anything able to read the user's files (malware, backup leak) | AutoCast must not copy this model. | Own a token-vault abstraction (encrypted at rest, OS keychain or master key). Do not port `.env` handling. |
| Medium | pendpost attestation key relies on POSIX file mode 0600; its test fails on Windows | `test/attest.test.mjs` ("the key file is mode 0600") | POSIX-only protection | Self-hosted on Windows would lack it | If ever ported: do not rely on POSIX modes; use OS keychain. |
| Medium | SimplePost IG publishing needs a public media URL; local files go to S3 (`S3MediaUploader`) | `utils/media-resolver.ts` imports `./s3`; `publishers/instagram/index.ts` `stageMedia` | Media sits in a bucket during processing; signed URLs may leak. SDK says it never logs the URL ("may carry signed query tokens"). | Adds an object store + AWS creds to the threat model for a single-user app | Prefer resumable `rupload` path (pendpost/Meta official) for local files; avoids a public bucket. |
| Low | pendpost YouTube upload is a single PUT of the whole file, no resume or status query | `scripts/yt-social.mjs:180-215` ("files are ~6 MB; no chunking needed") | Reliability, not security | Large videos fail and restart; risk of duplicate uploads if outcome is ambiguous | Use SimplePost's resumable-with-status-query logic or `googleapis` resumable instead. |
| Low | pendpost spawns child processes and `open`s URLs from API handlers | `lib/api.mjs:8,361-366,1979-2037` (`spawn(process.execPath, [scripts/…])`, `execFile('open', [url])`) | Arguments are fixed script names/URLs from server code; no shell string concat seen. Not exhaustive. | Low | If ported, keep argv arrays; never `exec` a string. |
| Informational | SimplePost SSRF guard is strong: private/loopback/link-local/cloud-metadata ranges blocked, DNS-lookup wrapper rejects hostnames that resolve privately, redirects capped and each hop revalidated | `utils/media.ts:83-232,371-383`, `ipaddr.js` | Defensive | Positive; reuse is valuable | Keep this behavior if media-by-URL is ever supported. |
| Informational | pendpost loopback hardening: binds 127.0.0.1 by default, rejects unknown `Host` (DNS rebinding), rejects foreign `Origin`, no-Origin requests allowed (MCP, curl) | `server.mjs:11,52-110`, SECURITY.md | Defensive. Caveat: `PENDPOST_HOST` can bind 0.0.0.0 (used in container). | Good pattern to copy for AutoCast's local API | Reproduce (Host + Origin allowlist). Add auth token for non-loopback binds. |
| Informational | pendpost traversal guards: `path.resolve` + `startsWith(allowedRoot)` in media route; filename validator (no dot-start, charset, extension allowlist); static server checks `startsWith(DIST + sep)` | `lib/media.mjs:26,54-55`, `lib/assets.mjs:77-84`, `lib/static.mjs:37-51` | Defensive | Good | Reference pattern. |
| Informational | pendpost approval gate: publish fail-closed without approval; creator cannot approve own post (owner exempt); Meta error 368 breaker halts Meta lane, never auto-resumes | SECURITY.md; `lib/accounts.mjs`, `lib/state.mjs`, `lib/auto-approve.mjs` | Defensive | Core safety design AutoCast should reimplement | REFERENCE → build on AutoCast schema. |
| Informational | pendpost duplicate protection is per-field persistence (`fbReelId`, `igMediaId`, `ytVideoId`) checked before each platform call | `scripts/meta-social.mjs:299,839,962,1005,1062` | Prevents repeats after a *recorded* success. Does not cover crash/timeout between platform success and local write. | See duplicate-publishing analysis in [REUSE_DECISION.md](REUSE_DECISION.md) | Build a write-ahead intent record. |
| Informational | SECURITY.md of both projects direct reports to private channels | `SECURITY.md` | n/a | Positive process signal | None |

## Not verified (UNKNOWN)

- SimplePost scheduler token encryption algorithm and key handling.
- SimplePost server auth, CSRF, API-key hashing, rate limiting, webhook handling.
- pendpost CSRF beyond Origin/Host checks; request body size limits; log redaction.
- Any dynamic testing. No exploit attempts were made.
- Whether pendpost's Playwright MCP engagement lane is reachable from publishing paths (it is pinned `@playwright/mcp@0.0.80` via `npx`, per `package.json`).

## Bottom line

No Critical finding. The only High is a dependency issue in a published npm release and looks already fixed on `main`. Neither project's token storage should be copied.
