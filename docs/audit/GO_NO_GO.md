# GO / NO-GO Gate — Phase 0.5

## Go-criteria check

| Criterion | Status | Basis |
|---|---|---|
| Licensing sufficiently understood | PASS with caveat | Repos and scanned tree: no GPL/AGPL. LGPL libvips and platform terms need **LEGAL** review only if redistributing / serving others. |
| No incompatible dependency selected | PASS | None selected. AGPL/unlicensed candidates rejected. |
| SimplePost/pendpost reuse technically viable | PASS | Model D; SimplePost PARTIALLY, pendpost ADAPT. |
| Official Meta path viable | PASS (live test pending) | Official docs verified; dev-mode publishing UNKNOWN. |
| Official YouTube path viable | PASS (live test pending) | Quota 1 unit/100 per day; unverified-project visibility UNKNOWN. |
| Queue choice understood | PASS | `pg-boss`; crash-recovery spike pending. |
| Duplicate-publishing architectural solution | PASS (design only) | State machine with write-ahead intent + UNKNOWN/reconcile in REUSE_DECISION. Reconcile reliability for YT/FB Page videos UNKNOWN. |
| Token/security model understood | PASS with caveat | Vault abstraction required; candidate crypto not reviewed. |
| Single-user self-hosted accepted | PASS (recommendation; user has not explicitly confirmed) | |
| No unnecessary LLM | PASS | |
| Browser automation not required | PASS | |
| Ownership boundary clear | PASS | |

## Conditions before / during Phase 1

1. **Choose AutoCast's license** (MIT or Apache-2.0) and add `LICENSE`. Add `THIRD_PARTY_NOTICES` for any ported pendpost code.
2. **User confirms** single-user self-hosted scope.
3. **Do not hard-depend on `@simple-post/sdk@1.5.0`**: wait for a release with patched `sharp`/`zod` or use overrides; CI gate on `npm audit` and a license scanner.
4. **Live spike** with the user's own accounts (outside the app architecture): (a) IG Reel via resumable upload in a Standard Access dev-mode app; (b) FB Page Reel; (c) YouTube upload from a fresh Cloud project, check visibility, `publishAt`, thumbnail, captions; (d) record Page-token lifetime. Update API_VERIFICATION with results.
5. **Queue spike**: pg-boss kill-worker/restart test and enqueue + intent-row in one transaction.
6. **Reconcile spike**: confirm how to detect a published-but-unacknowledged YT video and FB Page video without a marker; if not possible, accept `NEEDS_HUMAN` state.
7. **Token vault** design reviewed before any real token is stored; no plaintext `.env` secrets.
8. **Re-run tests on Linux/WSL** for both candidates before porting any code.
9. **Legal review** (professional) if AutoCast will be distributed as binaries/Docker image or offered to others (LGPL libvips, Meta/Google terms).

============================================================
AUTOCAST PHASE 0.5 — FINAL GATE
============================================================

Repository: https://github.com/anjinapp-aryan/AutoCast (main @ 1c61d4f, docs only)
Audit Date: 2026-10-27

License Status:        GREEN for chosen candidates; AutoCast license not yet chosen; LGPL libvips = LEGAL if redistributed
Security Status:       No Critical; 1 High (published SDK deps, appears fixed on main); candidate token storage must not be copied; deep review partial
Dependency Status:     Candidate stack scanned: 283 packages, 7 high (all via @simple-post/sdk@1.5.0), SBOM generated (CycloneDX)
Meta API Status:       Official path verified (v26.0 current); dev-mode publish + Page token lifetime UNKNOWN
YouTube API Status:    Official path verified; upload cost 1 unit/100 per day; unverified-project visibility UNKNOWN
Queue Status:          pg-boss preferred (MIT, Postgres-only); spike pending
Publishing Reliability Status: No platform idempotency keys; AutoCast-level state machine designed, not built
Self-Hosted Status:    Single-user self-hosted recommended
LLM Requirement:       NONE for MVP

SimplePost:
    PARTIALLY usable. ADAPT: YouTube resumable/thumbnail logic and retry classification; FB video upload. REFERENCE for IG (public-URL only). Not a hard core dependency yet.

pendpost:
    ADAPT: IG resumable Reel upload, FB Reels, YT captions, Meta 368 breaker. REFERENCE: approval gate, loopback hardening, per-field idempotency. REJECT: .env secrets, YT single-PUT upload, app/dashboard.

Final Architecture:
    Model D: AutoCast orchestration (Node/TypeScript, Postgres, pg-boss) + thin per-platform adapters over googleapis and official Graph REST, with ported MIT code.

AutoCast MUST BUILD:
    Campaign/content model, approval workflow, publish state machine + reconciliation + idempotency, audit log, token vault, adapter contract, account model, scheduling domain, thin API/CLI/UI.

AutoCast MUST NOT BUILD:
    OAuth protocol, Google API client, YouTube upload protocol, Meta API client library, job queue, MCP protocol, image processing, video transcoding.

Critical Risks:
    1. Duplicate publishing after ambiguous success (no platform idempotency).
    2. Platform gates: Meta App Review / YouTube audit if ever serving others.
    3. Published SimplePost release has vulnerable deps and heavy footprint.
    4. Candidates are young with ~2 contributors each.
    5. Token storage in candidates is plaintext or unreviewed.

Open Questions:
    1. AutoCast license?
    2. Confirm single-user self-hosted?
    3. "PostBox" repository: which was meant?
    4. Postgres required, or SQLite wanted?
    5. Are the user's IG/FB accounts professional and Page-linked?

FINAL VERDICT:
    GO WITH CONDITIONS

Conditions Before Phase 1:
    Items 1, 2 and 3 above before code; items 4–9 during Phase 1 (4–6 before the first live adapter ships).

============================================================
