# 07 — License Decision

Classification: GREEN = MIT/BSD/Apache; YELLOW = weak copyleft; RED = GPL/AGPL/proprietary/source-available/no license.

| Project | License (source) | Class | Direct reuse? | Adapt? | Reference? | Reject? | Reason |
|---|---|---|---|---|---|---|---|
| pendpost | MIT (`LICENSE`, `package.json`, npm `pendpost@2.7.0`) | GREEN | Possible but shaped as an app | **Yes** (port Reels/captions/breaker logic, keep copyright notice) | Yes | No | README says the repo is a "subset" synced from a closed site and a paid cloud tier exists; the MIT repo content is still MIT. `DISCLAIMER.md` is a project posture, not a license term. |
| SimplePost (`simple-post/core`) | MIT (root `LICENSE`; also `sdk`, `server`, `scheduler`, `cli` `package.json`) | GREEN | **Yes** (`@simple-post/sdk` from npm) | Yes | Yes | No | Hosted product exists; "published vs hosted behavior" differs (README). |
| Postiz | AGPL-3.0 (`LICENSE`) | RED | No | No | Architecture only | Code: **Yes** | Network copyleft would require publishing AutoCast source for network users if derived. Needs legal review for any use, even running it unmodified as a separate service. |
| Mixpost Lite | MIT (`LICENSE.md`) | GREEN | No (PHP; no YouTube) | No | Yes | No for ideas | Lite of a commercial product; maintainers state Lite features are kept distinct from Pro. |
| Open-Dispatch | MIT (GitHub license API) | GREEN | No | No | Yes | No | Too thin; 2 contributors. |
| RolePlayingTech/social-media-manager | **None** (no LICENSE file; GitHub `license: null`) | RED | **No** | **No** | Idea-level only | **Yes** | No license means no right to copy or adapt. |
| TryPost (`trypostit/trypost`) | AGPL-3.0 (GitHub API) | RED | No | No | No | Yes | Not code-audited. |
| Growchief | AGPL-3.0 (GitHub search metadata) | RED | No | No | No | Yes | Browser automation; not audited. |
| cogsend | MIT (`LICENSE`) | GREEN | Not audited | — | — | — | Cloudflare Workers based; out of scope of IG/FB/YT focus (not audited in depth). |
| fberrez/social-queue | ISC (`LICENSE`) | GREEN | No | No | No | — | Single maintainer, last push 2026-03-26; Bluesky/LinkedIn oriented (not audited in depth). |
| chronex | MIT (`LICENSE`) | GREEN | Not audited | — | — | — | Single maintainer, stale since 2026-05-27. |
| easy-uploader | MIT (`LICENSE`) | GREEN | No | No | No | Yes (for publishing) | Selenium/browser-driven (test scripts), plus many unrelated CI templates. |

## Dependency licenses

Verified with `npm view <pkg> license` on 2026-10-27 (direct dependencies and libraries AutoCast may adopt):

| Package | License | Class |
|---|---|---|
| `@simple-post/sdk` 1.5.0 | MIT | GREEN |
| `pendpost` 2.7.0 | MIT | GREEN |
| `googleapis` | Apache-2.0 | GREEN |
| `axios`, `form-data`, `cheerio`, `zod`, `uuid`, `ipaddr.js`, `@atproto/api` | MIT | GREEN |
| `sharp`, `twitter-api-v2`, `@aws-sdk/client-s3`, `prisma`, `@prisma/client` | Apache-2.0 | GREEN |
| `next` | MIT | GREEN |
| `pg-boss` 12.38.0, `bullmq` 6.5.0, `@modelcontextprotocol/sdk` 1.32.2 | MIT | GREEN |
| `facebook-nodejs-business-sdk` | **"Platform License"** | RED |

**Transitive dependency licenses: UNKNOWN.** No license scanner was run over lockfiles; doing so is a required gate before Phase 2 (e.g., `license-checker` / `pip-licenses` in CI). `sharp` bundles libvips (LGPL) as prebuilt binaries; confirm obligations. `ffprobe/ffmpeg` builds vary between LGPL and GPL; confirm before bundling. `@playwright/mcp` is launched by pendpost via `npx` and was not license-checked.

Also verify the AutoCast project's own license choice (repo currently has none). Using MIT/Apache-2.0 for AutoCast keeps all GREEN candidates compatible.
