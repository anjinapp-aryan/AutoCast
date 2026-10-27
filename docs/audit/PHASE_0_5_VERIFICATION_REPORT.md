# AutoCast Phase 0.5 — Verification Report

Date: 2026-10-27. Documentation only; no application code, no dependencies added to AutoCast. Companion files: [LICENSE_MATRIX](LICENSE_MATRIX.md) · [SECURITY_AUDIT](SECURITY_AUDIT.md) · [API_VERIFICATION](API_VERIFICATION.md) · [DEPENDENCY_AUDIT](DEPENDENCY_AUDIT.md) · [QUEUE_COMPARISON](QUEUE_COMPARISON.md) · [REUSE_DECISION](REUSE_DECISION.md) · [GO_NO_GO](GO_NO_GO.md) · [sbom/](sbom/).

## 1. Repository state (AutoCast)

| Item | Actual |
|---|---|
| Branch / history | `main`, one commit `1c61d4f` "docs: add audit reports" (pushed in this session). Remote `origin` = `anjinapp-aryan/AutoCast`. |
| Files | `docs/audit/*` only (Phase 0 reports, now plus Phase 0.5). |
| Package manager, dependencies, config, CI/CD, Docker, env files | **None exist.** |
| License file | **None.** AutoCast is currently "all rights reserved" by default. |

Phase 0 said the repo was empty; since then it received the Phase 0 docs commit. Nothing else changed.

## 2. Verification of Phase 0 conclusions

| Phase 0 claim | Phase 0.5 result |
|---|---|
| SimplePost MIT, SDK covers IG/FB/YT | Confirmed MIT. IG only via public URL (S3 for local files); no FB Reels; no YT captions. See REUSE_DECISION. |
| pendpost MIT, zero runtime deps, IG resumable + FB Reels + YT captions | Confirmed from source. YT upload is a single PUT, no resume (new finding). |
| Postiz, TryPost, Growchief AGPL-3.0 | Confirmed via GitHub API. |
| Mixpost Lite MIT, no YouTube, stale | Confirmed: MIT, no `youtube` hit in `src/`, last push 2026-03-16. Repo is `inovector/mixpost`. |
| Open-Dispatch MIT, thin | Confirmed MIT (`Matthew-Selvam/Open-Dispatch`, 18 stars). Not re-audited deeply; Python, REFERENCE only. |
| YouTube `videos.insert` quota ambiguous | Resolved: 1 unit, separate 100/day bucket (official). |
| Unverified projects forced private | **Contradicted by Google's current reference page.** Real behavior UNKNOWN. |
| IG limit 100 vs 50 | Meta's own pages conflict; use 50 and query the endpoint. |
| "Transitive licenses UNKNOWN" | Now scanned for the candidate stack: no GPL/AGPL; one LGPL (libvips binary) and two non-SPDX fields. |
| Tests not run | Run: SimplePost SDK 943/952 pass (9 Windows path failures); pendpost 346/386 pass (sampled failures Windows-specific). |
| `facebook-nodejs-business-sdk` "Platform License" | Not re-fetched; stays REJECT (use Graph REST). |
| RolePlayingTech/social-media-manager unlicensed | Not re-fetched; stays REJECT. |
| Browser-automation publishers rejected | Not re-audited; decision stands on policy grounds (section 8). |
| "PostBox" | Still NOT FOUND. |

## 3. Storage

Single-user self-hosted target.

| Data | Recommendation |
|---|---|
| Metadata, campaigns, posts, attempts, audit events | PostgreSQL (chosen with pg-boss; allows enqueue + intent-write in one transaction) |
| Tokens | Encrypted columns or OS keychain behind a vault interface; never plaintext `.env` |
| Media | Local filesystem directory (content-addressed names); object storage only if a platform later needs a public URL |

SQLite would be simpler to ship, but then `pg-boss` is out and a different queue is needed. For a single-user box, Postgres in one Docker container is acceptable. Final call is a Phase 1 condition. Avoid multi-tenancy, RBAC, billing, orgs.

## 4. Deployment model

**Single-user self-hosted MVP.** Reasons: both platforms allow publishing to your own accounts with Standard Access (Meta) and a personal OAuth client (Google) without review; multi-user SaaS triggers Meta Advanced Access (App Review + Business Verification) and a YouTube compliance audit, plus token-vault and tenant-isolation obligations. No requirement found that justifies SaaS.

## 5. Official APIs vs browser automation

**Official APIs only for publishing.** Browser automation carries ToS/account-ban risk, breaks on UI changes, collides with MFA/CAPTCHA, and is unnecessary because IG, FB and YT all have official publishing APIs. It may exist only as an explicitly opt-in experimental module outside the MVP.

## 6. LLM requirement

Not needed for MVP publishing, scheduling, or IG/FB/YT adapters (all deterministic API calls). Later optional plug-ins: title/description/hashtag generation, repurposing, platform metadata, campaign planning, analytics summaries, natural-language scheduling. Core engine must not import an LLM SDK.

## 7. Hardware

Publishing workload only; no GPU.

| Tier | CPU / RAM / disk | Notes |
|---|---|---|
| Development | any laptop, 8 GB RAM, Docker optional | |
| Local self-hosted MVP | 2 cores, 2–4 GB RAM, 20 GB+ disk | Node + Postgres |
| 24/7 home server | same; stable uplink | Upload bandwidth is the bottleneck (video to IG/FB/YT) |
| Small VPS | 1–2 vCPU, 2 GB RAM, 20–50 GB disk | Disk grows with retained media |
| Production | not needed for MVP | |

Estimates from stack reasoning; **not benchmarked.** AI/video generation is a separate workload and the only reason a GPU would ever matter. ffmpeg transcoding is CPU.

## 8. Remaining candidates from the brief

Mixpost Lite and Open-Dispatch stay REFERENCE. The brief asks for a search for "any newly discovered superior alternative": Phase 0 `gh search` plus this phase's repo lookups found none that is MIT, YouTube-capable and library-shaped. Not exhaustive. `cogsend`, `chronex` remain un-audited.

## 9. What stayed UNKNOWN

- Meta dev-mode publishing and Page-token lifetime (needs live test).
- YouTube unverified-project visibility (needs live test).
- Queue crash-recovery behavior (needs spike).
- SimplePost scheduler/server security; encryption algorithm.
- 36 of 40 pendpost test failures not inspected.
- Newer `@simple-post/sdk` release status (only 1.5.0 installed).
- Prisma/Drizzle/ffmpeg license trees.
- Legal opinion on LGPL libvips, platform terms.

Final gate: see [GO_NO_GO.md](GO_NO_GO.md).
