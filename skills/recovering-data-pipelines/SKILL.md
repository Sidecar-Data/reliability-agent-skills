---
name: recovering-data-pipelines
description: Executes tool-level pipeline recovery using each tool's own idioms — clearing stuck or failed orchestrator tasks (Airflow/Dagster/Prefect), re-syncing broken connectors after schema drift (Fivetran/Airbyte/Stitch), rerunning failed dbt jobs with correct state selection, recovering warehouse-side failures (Snowflake/BigQuery), backfilling gaps idempotently, and reconciling rejected reverse-ETL records (Hightouch/Census). Ships per-tool playbooks under references/. Use when a diagnosed failure needs hands-on recovery in a specific tool: a deadlocked DAG, a connector load that broke on a source change, a dbt job to rerun from the point of failure, a history gap to backfill, or reverse-ETL rejects to reprocess. NOT for deciding what went wrong (investigating-data-incidents), incident command and consumer-facing containment decisions (responding-to-data-incidents — which calls this skill for the mechanics), or first-time setup of connectors and DAGs (Builder persona).
metadata:
  persona: reliability
  author: sidecar
---

# Recovering Data Pipelines

**Core principle: state first, buttons second.** Every recovery tool offers a tempting retry button, and pressing it without knowing the run's actual state is how one incident becomes two — double-loads, skewed watermarks, orphaned locks. Read the tool's state (run status, sync cursor, incremental high-water mark), predict what the recovery action will do to the data, then act using the tool's own recovery idiom, never a hand-rolled workaround.

## When to use

- A diagnosed failure needs recovery in a specific tool: stuck/deadlocked orchestrator task, failed or drifted connector sync, failed dbt Cloud/core job, warehouse-side abort, rejected reverse-ETL batch
- A table needs history backfilled after an outage or an extended lookback
- `responding-to-data-incidents` chose "rerun" or "backfill" and delegates the mechanics here

**Do NOT use for:** root-causing — recovery without a diagnosis is the retry-button anti-pattern (route to `investigating-data-incidents`); the incident-command loop of contain/verify/communicate (`responding-to-data-incidents` owns that and calls into this skill); standing up new connectors, DAGs, or CDC (Builder's `onboarding-data-sources`); recurring performance tuning of retries/timeouts as a project (Builder).

## Step 0 — Read the state before touching anything

For whichever tool is involved (per-tool detail in the playbooks below):

| Question | Why it gates the action |
|---|---|
| What state does the tool believe the run/sync is in? | "Failed" vs "stuck-running" vs "zombie" take different recovery verbs; killing a healthy slow run creates the incident |
| Where is the progress cursor? (incremental watermark, sync cursor, last successful partition) | Determines whether a plain rerun resumes correctly or double-loads |
| Is the operation idempotent as configured? | Merge/delete+insert strategies rerun safely; append-only does not |
| What runs next on the schedule, and when? | The next scheduled run can collide with, or silently absorb, your manual recovery |
| What did the tool log at failure? | The tool's own error taxonomy (see playbook) maps to the sanctioned recovery verb |

And from the diagnosis: the damaged window (first-bad to last-bad) — every recovery action is scoped to it.

## Workflow

1. **Load the matching playbook** from `references/` — [dbt](references/dbt.md), [orchestrators](references/orchestrators.md) (Airflow/Dagster/Prefect), [connectors](references/connectors.md) (Fivetran/Airbyte/Stitch), [warehouse](references/warehouse.md) (Snowflake/BigQuery), [reverse ETL](references/reverse-etl.md) (Hightouch/Census). Unknown tool → apply this skill's generic contract and note the gap (see flywheel, below).
2. **Read the state** (Step 0 table) and write down the plan: current state → action → expected end state → expected data delta (row counts, affected partitions) → undo path. A recovery plan whose data delta you can't predict isn't a plan.
3. **Quiesce before mutating.** Pause the schedule (or confirm the next run is far enough away), take/verify locks the tool provides, and make sure no concurrent run is mid-flight over the same target. Manual recovery racing a scheduled run is a classic double-load.
4. **Recover with the tool's idiom** — the playbook's verb, not a workaround: clear-and-retry the failed task (not the whole DAG), resync the affected tables (not the whole connector), `dbt retry` / build from the failure point with state selection (not a full rebuild), reprocess the rejected batch (not re-sync everything). Full-refresh/full-resync is the *last* resort and only over the damaged scope.
5. **Backfills follow the idempotency contract**: delete-and-reload the exact damaged window or merge on unique keys; predict the row delta before executing; chunk long backfills so progress survives interruption and each chunk verifies before the next.
6. **Verify at the data layer, then the tool layer.** Tool says green *and* the damaged window reconciles (counts/aggregates vs source or pre-incident baseline) *and* the watermark/cursor points where the next scheduled run expects it. That third check is the one everyone skips — a recovered table with a corrupted cursor fails again tomorrow.
7. **Un-quiesce and hand back.** Resume schedules, confirm the next scheduled run passes, and report to `responding-to-data-incidents`: action taken, evidence of restoration, cursor state, anything left degraded.
8. **Feed the flywheel.** Recovery revealed a gap in a playbook (new failure mode, new tool, wrong assumption) → draft the playbook addition as part of the change. The library compounds across incidents.

## Standards

- **The tool's recovery idiom or nothing.** Hand-editing state tables, deleting metadata rows, or bypassing the tool's API voids its consistency guarantees — if the sanctioned verb can't fix it, escalate rather than improvise.
- **Scope discipline.** Recover the damaged window, the failed task, the affected tables — never "resync everything to be safe." Blast-radius of the recovery must not exceed the incident's.
- **Zombie discrimination.** "Running for 6 hours" is either genuinely slow or dead — check heartbeats/progress markers per playbook before killing; killing a live long run is self-inflicted.
- **Credentials and destructive resets are approval-gated.** Rotating credentials, full-refresh over consumer-facing tables, or anything that drops data goes to the human with the plan and undo path first.
- **Report cursor state, always.** Every recovery report states where the watermark/cursor ended up — it's the difference between "fixed" and "fixed until tonight's run."

## Common mistakes

| Mistake | Fix |
|---|---|
| Pressing retry before reading state | State first: run status, cursor, idempotency, next schedule |
| Rerunning an append-only load | Make it idempotent first (delete window / merge on keys) or you're converting a gap into duplicates |
| Full refresh as first resort | Scoped resync/retry of the damaged unit; full refresh is last and needs approval on big/consumer-facing tables |
| Manual recovery racing the scheduler | Pause or confirm clearance before mutating |
| Killing a slow-but-alive run | Check heartbeat/progress markers per playbook first |
| Verifying only "tool says green" | Reconcile the data *and* check the cursor/watermark position |
| Hand-editing tool metadata | Sanctioned verbs only; escalate if they don't suffice |
| Unbounded backfill | Chunked, verified per chunk, scoped to the damaged window |

## STOP if you're about to…

- retry or rerun without having read the run state, cursor position, and idempotency posture
- execute a full refresh / full resync when a scoped recovery exists
- run a backfill whose row-count delta you haven't predicted
- mutate pipeline state while a scheduled run could collide with it
- hand-edit a tool's internal state store or metadata tables
- resume the schedule without verifying the cursor points where the next run expects
