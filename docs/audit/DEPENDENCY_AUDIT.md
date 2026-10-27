# Dependency Audit — Phase 0.5

Date: 2026-10-27. Node v24.14.1, npm 11.11.0, Windows 11. Nothing installed into the AutoCast repo; consumer tree built in the scratchpad.

## Candidate projects: install / build / test

```text
Repository: simple-post/core
Commit: 957ed6a86ef47e7d46148f227bd551c855c53858 (2026-10-27)
Node version: v24.14.1 (repo engines: >=24.22.1 <25 — MISMATCH, ran anyway)
Package manager: yarn 4.9.2 via corepack (`yarn workspaces focus @simple-post/sdk`)
Install result: OK (13 s)
Build result: not run (tsc --noEmit: exit 0)
Test result: 943 passed, 9 failed, 43 suites (42 pass, 1 fail)
Lint result: eslint ran with no output shown (tail empty); treated as clean but not independently confirmed
Known failures: all 9 in tests/media.test.ts, downloadToTempFile. Expected "/tmp/simplepost_x.mp4", got "\tmp\simplepost_x.mp4". Windows path separator in test expectations, not a product defect. Server/scheduler/cli workspaces NOT tested.
```

```text
Repository: pendpost/pendpost
Commit: 42ac77b29f7720a912bdce1507b16a2a56fca74f (2026-10-14), 33 commits
Node version: v24.14.1 (engines >=20)
Package manager: npm; zero runtime dependencies
Install result: n/a (nothing to install)
Build result: not run (dashboard needs `cd app && npm install`)
Test result: 346 pass, 40 fail (386 test files executed one by one, bash, 60 s timeout each). `npm run check` itself aborts on Windows cmd ("f was unexpected at this time", POSIX for-loop).
Lint result: none configured beyond `node --check`
Known failures (sampled 4 of 40): attest.test: "key file is mode 0600" (NTFS has no POSIX modes); lane-block + mock-id-clear: ENOENT on `...\data\data\plans\...` (doubled path segment on Windows); driver.test: ERR_UNSUPPORTED_ESM_URL_SCHEME ("absolute paths must be valid file:// URLs"). The other 36 failures were not inspected: UNKNOWN whether all are Windows-specific.
```

Rerun on Linux/WSL before relying on either result. Both candidates were not modified.

## Candidate stack vulnerability scan (`npm audit`)

Installed: `@simple-post/sdk@1.5.0 pg-boss@12.38.0 bullmq@6.5.0 @modelcontextprotocol/sdk@1.32.2 googleapis@188.0.0 pg@8.23.1`. Result: 0 critical, **7 high**, 0 moderate. Raw: [sbom/candidate-stack-npm-audit.json](sbom/candidate-stack-npm-audit.json).

| Package | Via | Introduced by |
|---|---|---|
| `sharp` | libvips CVE-2026-33327, -33328, -35590, -35591 | `@simple-post/sdk@1.5.0` pins sharp 0.34.5 |
| `zod` | memory exhaustion on oversized constrained collections | sdk |
| `@atproto/api`, `common-web`, `lexicon`, `xrpc` | inherit zod issue | sdk (Bluesky, unused by AutoCast) |
| `@simple-post/sdk` | inherits all the above | direct |

Finding: the **source** at `main` already declares `sharp ^0.35.5` and `zod ^4.4.3`, so these look fixed upstream but are **not in the published 1.5.0**. Whether a newer npm release exists with these bumps: UNKNOWN (only 1.5.0 was installed). Zero vulnerabilities came from pg-boss, bullmq, MCP SDK, googleapis or pg. pendpost has no dependencies to audit.

Deprecation warnings during install: `prebuild-install@7.1.3` (unmaintained), `glob@10.5.0` (old).

## SBOM

CycloneDX 1.x, production deps only, 262 components: [sbom/candidate-stack.cdx.json](sbom/candidate-stack.cdx.json). Generated with `npx @cyclonedx/cyclonedx-npm --omit dev`. SPDX not generated.

## Direct dependency inventory

| Package | Version | License | Maintenance (GitHub / npm) | Notes |
|---|---|---|---|---|
| `@simple-post/sdk` | 1.5.0 | MIT | pushed 2026-10-26; 808 commits; 2 contributors | Depends on 14 runtime packages including `@aws-sdk/*` (3), `@atproto/api`, `twitter-api-v2`, `sharp`, `googleapis ^150`, `cheerio`, `zod`. |
| `googleapis` | 188.0.0 resolved | Apache-2.0 | active | SDK asks `^150`; AutoCast would pin its own. Large package. |
| `@modelcontextprotocol/sdk` | 1.32.2 | MIT | active | |
| `pg-boss` | 12.38.0 | MIT | pushed 2026-10-26, 4.1k stars | Node >=22.12. Deps: pg, cron-parser, rrule-temporal, serialize-error. |
| `bullmq` | 6.5.0 | MIT | pushed 2026-10-27, 9.5k stars | Needs Redis. Deps: tslib, semver, msgpackr, cron-parser, node-abort-controller. |
| `pg` | 8.23.1 | MIT | active | |

## UNKNOWN items

- Vulnerability scan of SimplePost's `server`/`scheduler`/`cli` workspaces: not done.
- OSV / GitHub Advisory cross-check: only `npm audit` was used.
- Prisma/Drizzle/ffmpeg wrapper trees: not evaluated.
- Whether the 36 uninspected pendpost test failures are Windows-only.
