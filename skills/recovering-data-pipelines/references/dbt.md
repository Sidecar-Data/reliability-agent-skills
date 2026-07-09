# Recovery playbook — dbt (Cloud and core)

*Last reviewed: 2026-07. Volatile details (CLI flags, API shapes) — verify against the installed version.*

## Reading state

- **Which node failed, and why**: `run_results.json` from the failed run (dbt Cloud: run artifacts tab / admin API; core: `target/`) — per-node status, timing, and the compiled error. The *first* error is the cause; later ones are usually skips cascading.
- **What was skipped**: everything downstream of the failure is `skipped`, not failed — the recovery set is "failed node + its skipped descendants," not the whole job.
- **Incremental state**: for incremental models, check the target table's max timestamp/key vs source — a failed run may have partially applied (dbt statements are per-model transactions, so a *model* is all-or-nothing on most warehouses, but a multi-model run can be half-done).
- **Concurrency**: is the job scheduled to fire again soon? Is another environment (CI) writing to the same schemas?

## Recovery verbs, in order of preference

1. **`dbt retry`** — reruns exactly the failed-and-skipped set from the previous run's `run_results.json`. The sanctioned "resume from failure point." Requires the artifacts from the failed run and an unchanged project state.
2. **State-selected rebuild** — `dbt build --select result:error+ result:skipped+ --state <failed-run-artifacts>` when retry isn't available, or `--select <failed_model>+` when you know the node. Never an unselected `dbt build` as recovery.
3. **`--full-refresh`, scoped** — only when the diagnosis says incremental state is corrupted (bad merge, mutated logic mid-history): `dbt build --select <model> --full-refresh`. On large/consumer-facing models this is the approval-gated option; predict cost and duration first.
4. **Backfill via variable-scoped runs** — projects with `var('start_date')`-style windowing: chunked runs over the damaged window. Projects without windowing vars: delete-and-reload the damaged partitions manually *through the model's own logic* (temporarily constrain the incremental predicate), never by hand-written INSERTs that bypass the model.

## Gotchas

- **Retrying a run whose *code* was the problem** re-fails identically — retry is for transient causes (warehouse timeout, concurrency, OOM); code defects need the fix first (that's `responding-to-data-incidents`' revert/fix decision, not a retry).
- **dbt Cloud job overlap**: a manual recovery run and the scheduled run can execute concurrently unless the job queues; check the job's run-slot behavior before launching.
- **Source freshness failures are not model failures** — a red freshness check means recover the *loader* (connectors playbook), not rerun dbt; rerunning dbt on stale sources produces fresh-looking stale marts.
- **Snapshots cannot be rerun for missed history** — a snapshot that didn't run at T has lost T's state permanently. Recovery is "resume from now + note the gap," never fabricate rows.
- **CI schemas**: `state:modified` comparisons against stale production artifacts select the wrong set — confirm which artifacts the state comparison uses before trusting the selection.

## Verify

- Failed + skipped set now `success` in the new `run_results.json`; no new errors introduced.
- For incrementals: damaged-window row counts and a key aggregate reconcile vs source or pre-incident baseline.
- Tests on the recovered subtree pass (`dbt test --select <failed_model>+`).
- Next scheduled run completes green (watch it — this is the "cursor points where the next run expects" check).
