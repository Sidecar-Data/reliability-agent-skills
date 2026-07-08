---
name: responding-to-data-incidents
description: Remediates diagnosed data incidents with the smallest safe change — containing bad data (quarantine, rollback, pausing jobs), then restoring correctness via rerun, revert, minimal fix, or idempotent backfill, then closing the loop with verification, stakeholder comms, postmortems, and runbooks. Use when a root-caused failure needs fixing: a stuck DAG or sync, a bad deploy to roll back, duplicate or late-arriving data, a broken ref, a backfill after an outage, or an incident needing comms and a postmortem. NOT for diagnosing what went wrong (investigating-data-incidents — remediation without a diagnosis is guessing), adding preventive coverage (authoring-data-quality-tests), or feature work discovered during the incident (Builder persona).
metadata:
  persona: reliability
  author: sidecar
---

# Responding to Data Incidents

**Core principle: contain, then cure — with the smallest reversible change.** While you engineer the fix, consumers are reading wrong numbers; stopping the spread beats elegance. Every remediation is chosen by asking "what is the least we can change that restores correctness, and how do we undo it if we're wrong?"

## When to use

- A diagnosed incident needs remediation: rerun, revert, minimal code fix, or backfill
- A bad build shipped wrong numbers and must be quarantined or rolled back
- A stuck orchestration task, failed connector sync, or rejected reverse-ETL batch needs recovery
- Duplicate or late-arriving data needs idempotent repair
- An active incident needs ownership: status comms, a postmortem, a runbook

**Do NOT use for:** finding the root cause (route to `investigating-data-incidents` first — this skill *starts* from a diagnosis); writing the preventive tests afterward (`authoring-data-quality-tests`); the refactor or feature the incident revealed you "should really do" (file it for Builder). If the fix requires an upstream producer to change their schema or timing, coordinating that change is Custodian's.

## Step 0 — Gather context before changing anything

- **The diagnosis**: root cause, origin class (source / ingestion / transform / wrong check), first-bad time, evidence chain. Missing or unverified → route back to `investigating-data-incidents`; remediating an undiagnosed incident is guessing with production data.
- From **Sidecar platform context**: blast radius (downstream models, exposures, consumers and their usage); job topology (which scheduled runs will touch the affected assets next — your containment deadline); how the pipeline handles reruns today (incremental strategies, unique keys, watermarks, snapshot schedules).
- From **the human**: tolerance for gaps vs wrongness (is missing data acceptable while we fix, or must stale-but-plausible data stay up?); who must approve consumer-facing actions; where incident comms go.

## Workflow

1. **Contain first.** Before engineering the cure: pause the job that would propagate the badness on its next run; quarantine consumer-facing wrong data (roll back to the last good build, restrict the affected mart, or annotate the dashboard); notify affected consumers with what's known. A one-hour-old incident with clean containment beats a perfectly fixed one that spread for six.
2. **Choose the smallest remediation that fits the origin class**, in escalating order of invasiveness:
   - **Rerun** — for transient failures (deadlock, timeout, OOM, flaky source). Precondition: the operation is idempotent, or you've made it so.
   - **Config/infra fix** — retries, timeouts, credentials, incremental-field or sync-mode corrections. No code semantics change.
   - **Revert** — for a bad deploy: revert the merged change and rebuild. Choose revert over fix-forward whenever the fix isn't both obvious and small — reverting is the known-good state.
   - **Minimal code fix** — the smallest diff that restores correct behavior. Not a refactor, not a cleanup, not "while I'm here."
   - **Backfill** — for gaps and corrupted history. The most invasive: it rewrites the past. Scope it to the damaged window only.
3. **Make reruns and backfills idempotent before running them.** Delete-and-reload the exact damaged window, or merge on unique keys — never blind re-append (that converts a gap incident into a duplicate incident). State the expected row-count delta *before* executing; investigate any surprise before proceeding.
4. **Verify restoration empirically.** Failing checks now pass; reconcile the repaired window against the source of record or a pre-incident baseline (counts, key aggregates); confirm downstream assets rebuilt clean and freshness recovered. "The job went green" is not verification — the job was green while it produced wrong numbers.
5. **Un-contain deliberately.** Resume paused jobs, restore quarantined assets, and tell consumers the data is trustworthy again — with the verified evidence, not just "it's fixed."
6. **Close the loop.** Blameless postmortem for real incidents: timeline, root cause, blast radius, what detection missed, action items with owners. Recurring failure → write or update the runbook. Coverage gap exposed → hand the specific gap to `authoring-data-quality-tests`. Fix-revealed tech debt → file for Builder; do not do it now.

## Standards

- **Never delete or loosen a check to make a run green** unless the diagnosis proved the check wrong — and then say so explicitly in the change.
- **Consumer-facing destructive actions need a human.** Rolling back shipped numbers, truncating a mart, overwriting history beyond the damaged window: present the plan, evidence, and undo path; let a human approve.
- **No firefight refactors.** The remediation diff contains only what restores correctness. Everything else is a follow-up note.
- **Every change ships through the delivery contract** — `sidecar/` branch, pushed for human review; never merged, never auto-merged by you. Warehouse-level containment you execute directly (with approval where required); repo changes always go through review.
- **Report faithfully.** If the backfill is partial, the rerun flaky, or verification incomplete — say exactly that. "Restored except X" is a fine outcome; "fixed" that isn't, is how the next incident starts inside this one.

## Common mistakes

| Mistake | Fix |
|---|---|
| Engineering the fix while bad data spreads | Contain first: pause, quarantine, notify — then cure |
| Fix-forward when revert was available | Revert is the known-good state; fix-forward needs the fix to be obvious *and* small |
| Blind re-append backfills | Delete-and-reload the damaged window or merge on keys; predict the row delta first |
| Deleting the failing test to unblock the run | Only the diagnosis can condemn a test; otherwise fix the data |
| Backfilling more history than was damaged | Scope to the broken window; rewriting healthy history adds risk for nothing |
| "Job is green" as verification | Reconcile against source of record or pre-incident baseline |
| Refactoring during the firefight | Smallest diff only; file the cleanup |
| Silent recovery | Consumers who saw bad numbers need to hear it's fixed, and what was wrong |

## STOP if you're about to…

- remediate without a verified diagnosis (→ `investigating-data-incidents`)
- run a non-idempotent rerun or backfill against production tables
- delete, disable, or loosen a failing check without diagnostic proof it's wrong
- execute a consumer-facing rollback or truncation without human approval
- merge your own fix, or declare recovery without reconciliation evidence
