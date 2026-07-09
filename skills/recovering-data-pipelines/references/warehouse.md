# Recovery playbook — Warehouses (Snowflake, BigQuery)

*Last reviewed: 2026-07. Retention windows and feature availability vary by edition/plan — verify on the account.*

## Reading state

- **What actually happened to the statement**: query history (Snowflake: `QUERY_HISTORY`; BigQuery: `INFORMATION_SCHEMA.JOBS`) — completed, failed, cancelled, or still running. A "failed" job whose write actually committed, and a "succeeded" one that wrote zero rows, both happen; the query history's row counts are the truth.
- **Transactional posture**: single-statement writes are atomic on both platforms; multi-statement scripts without explicit transactions can be half-applied. Identify exactly which statements of a failed script committed.
- **Time-travel budget**: how far back can you rewind the damaged table (Snowflake time travel retention; BigQuery time travel window)? This decides whether "restore the table to T" is an option — and it's a *decaying* option, so establish it early.
- **Locks and running jobs**: is something still holding the table (long merge, another recovery attempt)?

## Recovery verbs, in order of preference

1. **Time-travel restore of the damaged table** — when the table was correct at a known T before the bad write: Snowflake `CREATE TABLE ... CLONE ... AT (TIMESTAMP => T)` (clone-then-swap, never restore in place blind); BigQuery `FOR SYSTEM_TIME AS OF` into a staging table, verify, then swap. This is the cleanest "undo a bad build" and the backbone of quarantine/rollback containment.
2. **Rerun the producing job** — when the write failed cleanly (atomic statement, nothing committed): recover at the orchestrator/dbt layer (their playbooks), not by hand-running SQL that imitates the model.
3. **Surgical repair of a half-applied script** — complete or reverse the uncommitted remainder *through the same logic that produces the table* where possible; hand-written repair DML is the last resort and gets a review like any code change.
4. **Kill a runaway statement** — cancel via the platform's kill verb after confirming it's the incident's cause (or victim), not an unrelated workload. Note what the cancel rolls back.

## Gotchas

- **Time travel decays while you deliberate** — if a restore might be needed, clone the known-good state *now* (clones are cheap metadata operations) and decide later; retention expiring mid-incident converts a 1-minute fix into a backfill project.
- **Swap, don't overwrite**: restore into a clone/staging table, reconcile it, then atomic-swap (`ALTER TABLE ... SWAP WITH` / table replace). Restoring blind over the live table with the wrong T makes a second incident.
- **Restored tables vs incremental state**: after restoring a table to T, downstream incremental models and connector cursors still believe post-T data exists — reconcile the watermarks (dbt/connector playbooks) or tomorrow's run re-corrupts it.
- **Clone/restore and access grants**: recreated tables can drop grants or inherit different ones — re-verify permissions after a swap; a "fixed" table nobody can read is still an incident.
- **Cost-shaped failures** (OOM/spill on Snowflake, slot exhaustion or bytes-scanned kill on BigQuery): the *recovery* is rerunning under the right warehouse size/slot reservation once; the *tuning* is Optimization persona work — do the first, file the second.

## Verify

- Restored/repaired table reconciles vs the known-good reference (counts, key aggregates, sampled rows) — not just "exists."
- Grants intact; downstream views/models still resolve.
- Watermarks/cursors of everything that writes to or reads incrementally from the table are consistent with its restored state.
- The next scheduled producing run completes green and doesn't reintroduce the damage.
