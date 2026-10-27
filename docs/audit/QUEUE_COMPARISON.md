# Queue Comparison — Phase 0.5

Date: 2026-10-27. Sources: `npm view`, GitHub API (license/activity). Feature columns come from the libraries' documented design as I know it; **I did not read their source or run spikes this phase**, so reliability claims are UNKNOWN until Phase 1 spike.

| Queue | License | Storage | Redis Required | Scheduling | Retry | Reliability | Complexity | Recommendation |
|---|---|---|---|---|---|---|---|---|
| `pg-boss` 12.38.0 | MIT | PostgreSQL only | No | Delayed jobs, cron (`cron-parser`), RRULE (`rrule-temporal`) | Yes, with backoff | Postgres transactions + `SKIP LOCKED`; durable across crashes (design claim, not spiked) | Low if Postgres already present; Node >=22.12; 4 direct deps | **Preferred** |
| `bullmq` 6.5.0 | MIT | Redis | **Yes** | Delayed jobs, repeatable/cron | Yes | Mature, 9.5k stars, active; durability depends on Redis persistence config | Medium: second stateful service | Alternative |
| `graphile-worker` 0.18.0 | MIT | PostgreSQL | No | Cron + delayed (`run_at`) | Yes | Postgres-backed; 2.4k stars; pushed 2026-10-20 | Low; Node >=22.18 | Alternative, not evaluated further |
| SQLite-backed queues | n/a | SQLite | No | Not searched | | | | UNKNOWN, not evaluated |

## Decision

Choose **`pg-boss`**, conditional on PostgreSQL being the store (see below). Reasons: one fewer service than BullMQ (no Redis), MIT, active (pushed 2026-10-26), supports delayed/cron jobs natively, and queue state lives in the same database as publishing attempts, which lets AutoCast enqueue a job and write its publish-intent row in **one transaction**. That property directly helps the duplicate-publishing problem.

Evidence limits:
- Crash recovery, exactly-once semantics and lock behavior are library claims; verify with a Phase 1 kill-the-worker test.
- A queue is not a substitute for AutoCast's own publish state machine. At-least-once job delivery means the publish handler must be idempotent regardless of queue.
- If AutoCast ends up on SQLite (single-file MVP), pg-boss does not apply; fall back to a Postgres-in-Docker or BullMQ+Redis. This is why storage and queue are decided together.

Do not build a custom queue.
