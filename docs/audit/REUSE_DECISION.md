# Reuse Decision — Phase 0.5

Date: 2026-10-27. Evidence: source reads of `simple-post/core@957ed6a` and `pendpost@42ac77b`, official docs ([API_VERIFICATION.md](API_VERIFICATION.md)), dependency scans ([DEPENDENCY_AUDIT.md](DEPENDENCY_AUDIT.md)).

## 1. Can AutoCast use SimplePost as a backend adapter without rewriting its platform implementation?

**PARTIALLY.**

| Capability | `@simple-post/sdk` (source `sdk/src/publishers/*`) | Evidence |
|---|---|---|
| IG image / video / Reel | YES, but **only from a public URL**. Standalone video becomes `REELS`; carousel video child is `VIDEO`. | `instagram/index.ts` `createMediaObject` |
| IG local file | Only via **S3 upload** to get a public URL (`S3MediaUploader`). No `upload_type=resumable` / `rupload` path (grep: no `rupload` in SDK). | `utils/media-resolver.ts`, `instagram/index.ts` `stageMedia` |
| IG container status polling | YES: 10-minute deadline, retries transient reads on same container, ERROR/EXPIRED fail immediately | `waitForMediaReady` |
| IG publish retry safety | YES, careful: retries `media_publish` only for code 9007/subcode 2207027 ("not visible yet", rejected before publish); any other failure is thrown and not auto-retried | `publishContainer` |
| IG container retry | YES: container creation is retried since an orphan container expires | `createMediaObject` |
| IG permalink | YES, 5 attempts | `fetchPermalinkWithRetry` |
| FB page video | YES: start/transfer/finish chunked upload from local file, or `file_url`; `scheduled_publish_time` supported; ambiguous chunk failure abandons rather than publishing | `facebook/video-upload.ts` |
| FB Reels (`/video_reels`) | **NO** (no `video_reels` string in `publishers/facebook`) | grep |
| YouTube resumable upload | YES: own implementation (`resumable-upload.ts`), local file stream, 308/Range handling, `bytes */size` status query | `youtube/resumable-upload.ts:254-318` |
| YouTube title/description/tags/category/privacy/`publishAt`/kids flag | YES; `publishAt` forces `privacyStatus=private` | `youtube/index.ts:195-245` |
| YouTube thumbnail | YES via `thumbnails.set` | `youtube/index.ts:280` |
| YouTube captions | **NO** | grep |
| Platform validation | YES per platform | `validation.ts` files |
| Idempotency | SDK: **none**. `idempotencyKey` exists only in the hosted server API schema (`types/api.ts:172`), not the SDK | grep |
| Footprint | Heavy: AWS SDK, atproto, twitter-api-v2, sharp, cheerio, googleapis for 3 platforms | `sdk/package.json` |

Why PARTIALLY: YouTube and FB video are usable as-is; Instagram forces a public-URL/S3 detour for local media; FB Reels and YouTube captions are missing; published 1.5.0 carries high-severity audit findings and a large unrelated dependency tree.

## 2. pendpost as a source of patterns

| Piece | Verdict | Evidence / reason |
|---|---|---|
| IG resumable Reel upload to `rupload.facebook.com/ig-api-upload/v24.0`, container creation with `upload_type=resumable`, `media_publish` | **ADAPT** | `scripts/meta-social.mjs:79,1066,1113`. Matches the official flow. Embedded in a ~1,700-line CLI script with filesystem plans; needs extraction. |
| FB Reels 3-phase (`video_reels` start → rupload → finish) | **ADAPT** | `meta-social.mjs:201-243`; matches official docs. |
| YouTube captions (`captions.insert`, multipart SRT) | **ADAPT** (~25 lines) or call `googleapis` | `yt-social.mjs:387-410`. Needs `youtube.force-ssl` scope. |
| YouTube upload | **REJECT** | Single PUT with no resume or status query (`yt-social.mjs:180-215`). SimplePost's is better. |
| YouTube thumbnail | REFERENCE | `yt-social.mjs:361-382`; SimplePost already covers it. |
| Meta 368 circuit breaker | **ADAPT** (rule is small; storage is file-based) | `lib/accounts.mjs` |
| Approval gate, no self-approval | **REFERENCE** → build on AutoCast schema | SECURITY.md, `lib/auto-approve.mjs` |
| Per-field idempotency (`igMediaId`, `fbReelId`, `ytVideoId`) | **REFERENCE** | Good start; has a crash window (below). |
| Loopback + Host/Origin hardening | **REFERENCE / copy pattern** | `server.mjs:52-110` |
| `.env` secret storage | **REJECT** | Plaintext secrets. |
| Dashboard, Playwright engagement lane, radar/agent features, 20 other platforms | **REJECT** | Out of scope; browser automation is not for publishing. |
| Whole app as AutoCast base | **REJECT** | File-based, single-user, subset of a commercial product; repo has `DISCLAIMER.md` and a cloud tier. |

License for ported pieces: MIT; keep the copyright and permission notice.

## 3. Duplicate-publishing analysis

Scenario: AutoCast calls `media_publish` / `videos.insert`; platform succeeds; response is lost.

Official platform facts (see API_VERIFICATION): **no idempotency key** on `media_publish`, `video_reels` or `videos.insert`. Instagram: container `status_code=PUBLISHED` is queryable. YouTube: only the live resumable session URI can replay the completion response. Facebook: no documented dedupe.

What candidates provide:
- SimplePost SDK: classifies errors and refuses to auto-retry ambiguous `media_publish` failures. Good, but gives the caller no recovery.
- pendpost: records the platform id after success and skips that platform next run. Crash/timeout between platform success and local write re-publishes.

Neither closes the gap. **AutoCast must own this.** Recommended AutoCast-level state machine (design only):

```text
DRAFT -> APPROVED -> SCHEDULED -> CLAIMED
CLAIMED --(write publish-intent row BEFORE any platform call, in the same DB txn as the queue job)--> PREPARING
PREPARING (upload / container / poll; safe to retry) --> READY_TO_COMMIT   [persist containerId / sessionUri / videoId]
READY_TO_COMMIT --(set commit_started_at, attempt_n)--> COMMITTING        [the only non-repeatable call]
COMMITTING --ok + platform id--> SUCCEEDED
COMMITTING --definite 4xx rejected before publish--> FAILED
COMMITTING --timeout / network / 5xx / crash--> UNKNOWN
UNKNOWN --reconcile--> SUCCEEDED | FAILED | NEEDS_HUMAN
```

Reconcile rules per platform:
- Instagram: `GET /<containerId>?fields=status_code`; `PUBLISHED` → SUCCEEDED (look up media id via recent media + caption/time match); `FINISHED` after cool-down → safe to retry once; `EXPIRED`/`ERROR` → FAILED.
- YouTube: if `sessionUri` stored, empty PUT `bytes */SIZE`; else search own channel uploads by a unique marker (e.g. title/description token) using `videos.list` (1 unit) — **UNKNOWN whether this is reliable; verify in spike**.
- Facebook Reels: `GET /<video-id>?fields=status` (documented for Reels); Page videos: list recent Page videos and match marker — UNKNOWN reliability.

Rule: never auto-retry from `COMMITTING`/`UNKNOWN`; only reconcile. Embed a unique marker per post attempt in caption/title only if the user accepts it; otherwise `NEEDS_HUMAN`.

## 4. Ownership boundary (verified)

MUST NOT BUILD: OAuth protocol (use `googleapis`, Meta OAuth endpoints directly), Google API client, YouTube resumable upload protocol (use `googleapis` or SimplePost's helper), Meta API client wrapper beyond thin adapters, job queue (`pg-boss`), MCP protocol (`@modelcontextprotocol/sdk`), image processing, video transcoding (ffmpeg binary).

AUTOCAST OWNS: campaign/content model, approval workflow, publish state machine + reconciliation + idempotency, audit log, token vault abstraction, adapter contract, account model, scheduling domain, thin API/CLI/UI, provider-independent orchestration.

Verified: nothing in either candidate provides the state machine, vault or reconciliation; those are legitimately AutoCast-owned.

## 5. Architecture models

Scores 1–5 (higher better; Complexity 5 = least complex). Subjective, based on the evidence above.

| Model | Reuse | License | Maintenance | Flexibility | Reliability | Complexity | Score |
|---|---:|---:|---:|---:|---:|---:|---:|
| A. Build everything | 1 | 5 | 2 | 5 | 2 | 1 | 16 |
| B. Use SimplePost directly | 3 | 5 | 3 | 2 | 3 | 3 | 19 |
| C. Use pendpost directly | 3 | 5 | 3 | 1 | 3 | 3 | 18 |
| **D. AutoCast orchestration + adapters over libraries (`googleapis`, official Graph REST, selected SimplePost/pendpost code)** | 4 | 5 | 4 | 5 | 4 | 3 | **25** |
| E. Fork existing app | 4 | 3 | 2 | 2 | 3 | 2 | 16 |

**Recommendation: Model D.** Adapter specifics:
- YouTube: `googleapis` (Apache-2.0) for OAuth/captions/thumbnails; borrow SimplePost's resumable-with-status-query ideas or depend on the SDK for this one adapter if Phase 1 spike shows it clean.
- Facebook Page video/Reels and Instagram: small direct Graph REST adapter, ported from pendpost (`rupload`, `video_reels`) with MIT notice, informed by SimplePost's retry classification.
- Decision on depending on `@simple-post/sdk`: **do not make it a hard core dependency** in Phase 1 (heavy tree, IG public-URL limit, vulnerable published release). Revisit once a patched release exists; allow it behind the adapter contract.
