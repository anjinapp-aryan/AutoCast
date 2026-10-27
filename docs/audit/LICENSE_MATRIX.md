# License Matrix — Phase 0.5

Date: 2026-10-27. Method: `gh api repos/...` (SPDX), `LICENSE` files in fresh clones, `npm view`, `license-checker-rseidelsohn` over an installed consumer tree (`@simple-post/sdk@1.5.0`, `googleapis`, `@modelcontextprotocol/sdk`, `pg-boss`, `bullmq`, `pg`; 283 packages incl. platform-specific binaries). Raw output: [sbom/candidate-stack-licenses.json](sbom/candidate-stack-licenses.json).

GREEN = compatible. YELLOW = needs review. RED = do not use. This is an engineering reading, not legal advice. Items marked **LEGAL** need a professional opinion before distribution.

## Repositories

| Repo | License (verified) | Class | Note |
|---|---|---|---|
| simple-post/core | MIT, "Copyright (c) 2026 Vladimir Haltakov" | GREEN | Also MIT in `sdk/package.json`. |
| pendpost/pendpost | MIT, "Copyright (c) 2026 Nomadik GmbH" | GREEN | README says repo is a subset of a commercial site; MIT applies to what is in the repo. |
| Matthew-Selvam/Open-Dispatch | MIT | GREEN | Python; thin. |
| inovector/mixpost (Lite) | MIT | GREEN | PHP/Laravel; no YouTube in `src/`; last push 2026-03-16. |
| gitroomhq/postiz-app | AGPL-3.0 | RED | Confirmed via GitHub API. |
| trypostit/trypost | AGPL-3.0 | RED | Confirmed. |
| growchief/growchief | AGPL-3.0 | RED | Confirmed. |
| RolePlayingTech/social-media-manager | none (Phase 0) | RED | Not re-fetched this phase. No license means no reuse right. |
| facebook-nodejs-business-sdk | "Platform License" (Phase 0, npm) | RED | Not re-fetched. Call Graph API over HTTP instead. |

## Dependency inventory result (candidate stack)

| License | Count | Class |
|---|---:|---|
| MIT | 194 | GREEN |
| Apache-2.0 | 46 | GREEN |
| ISC | 19 | GREEN |
| BSD-2-Clause / BSD-3-Clause | 14 | GREEN |
| BlueOak-1.0.0 | 4 | GREEN (permissive) |
| 0BSD, (MIT OR CC0-1.0), (Apache-2.0 AND MIT) | 3 | GREEN |
| `Apache*` (twitter-text@3.1.0) | 1 | YELLOW: license field is non-SPDX; confirm file text. Reached only via `@simple-post/sdk`. |
| `BSD*` (url-template@2.0.8) | 1 | YELLOW: non-SPDX field; confirm file text. Via `googleapis`. |
| **Apache-2.0 AND LGPL-3.0-or-later** (`@img/sharp-win32-x64@0.34.5`, libvips binary) | 1 | **YELLOW / LEGAL**: dynamic LGPL library inside an npm binary package. Fine for self-hosting; redistribution (Docker image, installer) carries LGPL obligations. |

No GPL, AGPL or "no-license" package found in the scanned tree. Transitive licenses for pendpost: it has zero runtime npm dependencies (`package.json` has no `dependencies`), so there is nothing to scan beyond its own source. Transitive licenses for Prisma, ffmpeg wrappers, Drizzle: **UNKNOWN** (not installed; not selected).

## Compatibility decisions

| Component | License | AutoCast compatible? | Conditions | Recommendation |
|---|---|---|---|---|
| A. depend on `@simple-post/sdk` | MIT | GREEN legally | Keep MIT notice. Pulls `sharp` (LGPL libvips binary, **LEGAL** if redistributed), `twitter-text` (`Apache*`, YELLOW), plus AWS, atproto and twitter clients AutoCast would never call. Published 1.5.0 has 7 high `npm audit` findings (see [DEPENDENCY_AUDIT.md](DEPENDENCY_AUDIT.md)). | YELLOW: legally fine, technically heavy. See [REUSE_DECISION.md](REUSE_DECISION.md). |
| B. copy/port from pendpost | MIT | GREEN | Retain copyright + permission notice in each ported file or a `THIRD_PARTY_NOTICES`. Do not copy pendpost trademark/branding. `DISCLAIMER.md` is a posture, not a license term. | GREEN |
| C. depend on `googleapis` | Apache-2.0 | GREEN | Ship NOTICE if present. | GREEN |
| D. depend on `@modelcontextprotocol/sdk` | MIT | GREEN | None. | GREEN |
| E. use Meta Graph API directly | Meta Platform Terms / Developer Policies (not an OSS license) | YELLOW | Must accept Meta Platform Terms; app review for third-party accounts; no copying of Meta SDK code under "Platform License". | YELLOW: allowed, with terms compliance. **LEGAL** only if AutoCast is ever offered to other people's accounts. |
| F. use YouTube Data API directly | YouTube API Services Terms + Google API ToS | YELLOW | Must comply with API Services Terms of Service; quota extension needs compliance audit. | YELLOW |
| G. any queue | `pg-boss` MIT, `bullmq` MIT, `graphile-worker` MIT | GREEN | None. | GREEN |
| `pg` (node-postgres) | MIT | GREEN | | GREEN |
| ffmpeg (binary, not in npm tree) | LGPL or GPL depending on build | YELLOW / **LEGAL** | Use a dynamically-invoked system binary; do not bundle a GPL build. | Decide at Phase 3. |
| Prisma / Drizzle | Apache-2.0 (Phase 0) / not verified | UNKNOWN | Not installed this phase. | Re-check if chosen. |

## Open license risks

1. AutoCast has no license of its own yet. Choose MIT or Apache-2.0 before Phase 1 code lands.
2. `sharp`/libvips LGPL: only matters if AutoCast distributes binaries or images. **LEGAL.**
3. Terms of Meta and Google platforms are not OSS licenses; they govern use, not code. Not legal-reviewed.
