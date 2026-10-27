# 05 — Proposed Architecture

Model chosen: **D (build only the orchestration layer) implemented through B (existing publishing code behind an adapter)**, with queue and MCP composed from libraries. Reasoning in [FINAL-REPORT.md](FINAL-REPORT.md). This is a proposal; nothing is implemented.

```
User (dashboard / CLI / MCP client / optional AI agent)
        │
        ▼
AutoCast API  ── auth, audit log (append-only)
        │
        ├── Content & Campaigns   (AutoCast domain: BUILD)
        │        │
        │        ▼
        ├── Approval gate         (draft → approved | rejected; creator ≠ approver; fail-closed)
        │        │
        │        ▼
        ├── Scheduler / queue     (pg-boss or BullMQ: COMPOSE; per-account lock; due-time tick)
        │        │
        │        ▼
        └── Publisher abstraction  (interface: validate / publish / status / reconcile)
                 │
      ┌──────────┼───────────┐
      ▼          ▼           ▼
  Instagram   Facebook    YouTube          ← adapters
  adapter     adapter     adapter
      │          │           │
      └── existing MIT publishing code (@simple-post/sdk, googleapis,
          ported pendpost Reels/captions logic) ──┘
                 │
                 ▼
   Official APIs only: Meta Graph API · YouTube Data API v3 · OAuth 2.0

Side components
  Token vault   – encrypted at rest (BUILD thin, on Node crypto/libsodium)
  Media store   – local disk first, S3-compatible later; resumable upload from disk
  AI layer      – OPTIONAL plug-in producing drafts only; null implementation by default
  MCP server    – composed from @modelcontextprotocol/sdk; same contract as the REST API
```

## Rules

1. A `PublisherAdapter` interface isolates every external engine. Swapping `@simple-post/sdk` for something else must touch one folder.
2. Publish state machine: `scheduled → publishing → published | failed | unknown`. Failures after the irreversible call (`media_publish`, YouTube upload completion) map to `unknown` and are reconciled by querying the platform, never blindly retried (pattern from SimplePost IG publisher).
3. Native platform scheduling is used only if it can be reconciled; otherwise AutoCast's own queue fires at due time.
4. Mock adapters implement the same interface for Phase 2 (local MVP) and tests.
5. Browser automation is not a supported adapter type.
