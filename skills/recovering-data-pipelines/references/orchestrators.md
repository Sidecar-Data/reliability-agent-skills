# Recovery playbook — Orchestrators (Airflow, Dagster, Prefect)

*Last reviewed: 2026-07. UI/API surfaces move fast — verify verbs against the installed version.*

## Reading state (all three)

- **Task-level state, not DAG-level**: which task failed vs which are queued/upstream-failed. Recovery targets the failed task; the rest resolve by dependency.
- **Zombie discrimination**: "running" with no heartbeat/log progress ≠ running. Airflow: task heartbeat + worker liveness (zombie detection reaps these eventually). Dagster: run worker health / step heartbeats. Prefect: flow-run heartbeat + work-pool worker status. A run that stopped logging N minutes ago on a task that normally logs continuously is dead; a silent-but-progressing warehouse query is not — check the warehouse side before killing.
- **What the task does on rerun**: is the task idempotent (delete+insert its own window, merge, or pure trigger of an idempotent downstream)? A task built on data-interval/logical-date templating usually reruns cleanly *for its own interval*; a task using "now()" does not.
- **Scheduler pressure**: is catchup/backfill going to enqueue more runs the moment you unpause?

## Recovery verbs

- **Airflow**: *Clear* the failed task instance (with or without downstream) — the scheduler reruns it for the same data interval. Whole-run rerun only when the run itself is corrupt. Backfill = clearing a date range of task instances, or the backfill command with the interval bounds; keep catchup semantics in mind (`catchup=False` DAGs won't self-heal missed intervals — clear explicitly).
- **Dagster**: *Re-execute from failure* (only failed steps and their downstream re-run, honoring the same partition/run config). Partitioned assets: launch a **backfill over the exact damaged partition range**, not a full-asset backfill.
- **Prefect**: *Retry* the flow run from the failed task where supported, else re-trigger the flow with the same parameters for the damaged window. Confirm the deployment's schedule won't double-fire.

Order of preference everywhere: **clear/retry the failed unit for its original interval** → **re-execute from failure** → **scoped backfill of the damaged interval range** → whole-DAG/asset rerun (last, and only with idempotency confirmed).

## Gotchas

- **Deadlocks and pool starvation**: a "stuck" task may be waiting on a pool slot/concurrency limit, not broken — check pool/slot occupancy before clearing; clearing doesn't help a starved task and can worsen the queue.
- **Clearing with downstream on a fan-out DAG** can enqueue an enormous set — count what a clear will schedule before confirming it.
- **Sensor/trigger tasks**: rerunning a sensor whose condition already passed hangs forever, or fires instantly against the wrong state — check what the sensor actually senses.
- **Rerun-for-original-interval is the whole game**: rerunning "for today" a task that failed "for last Tuesday" writes the wrong window. Always rerun with the original logical date/partition, which is what the sanctioned verbs do and ad-hoc re-triggers often don't.
- **Kill before clear on zombies**: clearing a task whose old process is still secretly alive gives two writers on one target. Confirm the old attempt is dead (worker gone, warehouse query cancelled) first.

## Verify

- The cleared/re-executed tasks show success **for the damaged intervals/partitions**, not just the latest.
- Downstream tasks resolved by dependency (no stranded `upstream_failed`).
- Data written for the recovered intervals reconciles (counts/aggregates).
- Scheduler resumed; the next natural interval runs green without manual help; no unexpected catchup queue formed.
