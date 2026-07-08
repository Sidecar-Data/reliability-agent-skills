---
name: investigating-data-incidents
description: Diagnoses data incidents from symptom to root cause — triaging failed dbt jobs, freshness stalls, volume anomalies, and monitor breaches by walking lineage upstream, bisecting to the first bad point in time, and classifying the origin (bad source data, ingestion failure, transform logic, or a wrong test/monitor). Use when a monitor fires, a dbt job fails, a dashboard shows wrong or stale numbers, warehouse data disagrees with the system of record, or someone reports "the data looks off." NOT for implementing the fix (responding-to-data-incidents), adding preventive test coverage (authoring-data-quality-tests), building new models (Builder persona), or reviewing an upstream team's proposed breaking change (Custodian persona).
metadata:
  persona: reliability
  author: sidecar
---

# Investigating Data Incidents

**Core principle: evidence before hypothesis.** A diagnosis is a causal chain from root cause to symptom in which every hop is verified with a query, a log, or a diff — not inferred from what usually breaks. The most expensive investigation mistake is fixing the first plausible suspect; the second most expensive is trusting an alert's framing of what's wrong.

## When to use

- A freshness, volume, or custom SQL monitor fires
- A dbt job fails and someone needs to know why (not just which model)
- A stakeholder reports a broken dashboard or a number that "looks wrong"
- Warehouse data disagrees with the system of record (reconciliation)
- An open incident needs a root cause and an origin classification

**Do NOT use for:** executing the remediation — backfill, rerun, revert, quarantine (hand the diagnosis to `responding-to-data-incidents`); adding tests to prevent recurrence (`authoring-data-quality-tests`); anything that starts as a design request rather than a symptom (Builder). If diagnosis reveals an upstream producer changed a schema deliberately, the negotiation belongs to Custodian — deliver the finding and route.

## Step 0 — Gather context before forming any hypothesis

From **Sidecar platform context** (fall back to dbt artifacts + warehouse queries if unavailable):

| Question | Where to look |
|---|---|
| What exactly fired, and what is its threshold/config? | Monitor/test definition + its run history — is this test chronically red or newly red? |
| What is upstream of the symptomatic asset? | Lineage, column-level where the symptom is one column |
| What is downstream — who is consuming wrong data right now? | Lineage + exposures + usage (dashboard popularity, query activity) |
| What changed recently? | dbt run history, merged PRs touching the path, connector/sync logs, source schema drift |
| When was it last verifiably right? | Test pass history, snapshot/audit tables, warehouse time travel if available |

From **the human** (ask only what changes the investigation):
- What should the number be, and on what authority? ("Finance says revenue is $2.1M" beats "it looks low")
- When did it last look right, and how tolerant are consumers of the current badness? (drives containment urgency, not diagnosis)

## Workflow

1. **Pin the symptom precisely.** One sentence: which asset, which column/metric, expected vs observed, first-seen time. If you cannot reproduce the symptom with a query, stop — you may be chasing a stale alert or a caching artifact in the BI layer.
2. **Establish blast radius immediately** (before root-causing). From lineage: downstream models, exposures, and who queried the affected assets recently. Report it — containment decisions (pause the job? warn consumers?) belong to `responding-to-data-incidents`, but they need the radius *now*, not after the diagnosis.
3. **Bisect in time.** Find the first bad partition/run/day: compare row counts, key aggregates, or checksums across time until you have "last good, first bad." A first-bad timestamp converts "something is wrong" into "what happened between T1 and T2" — a vastly smaller search space.
4. **Walk lineage upstream, verifying each hop.** At every parent, run the cheapest query that answers "is the badness already present here?" Bad at parent → move up. Clean at parent, bad at child → the defect is in that child's transform. Do not skip hops on intuition; fan-out and filter bugs hide in "obviously innocent" models.
5. **Correlate the origin with a change event.** At the origin node, find what happened at the first-bad time: a merged PR, a connector sync failure or schema drift, a source-side deploy, a backfill, a late-arriving batch. No matching change event → widen the window or question the bisection, not the evidence.
6. **Classify the origin — exactly one primary:**
   - **Bad source data** — the producer sent wrong/absent data; warehouse faithfully reflects it
   - **Ingestion/ETL failure** — connector stall, sync error, schema drift, duplicate/late loads
   - **Transform defect** — model logic, a bad merge, a broken ref
   - **The test/monitor is wrong** — data is correct; the check's threshold, freshness SLA, or assumption is stale
   The last one is a first-class outcome, not an excuse: prove it by validating the data against an independent authority, not by observing that the test is annoying.
7. **Deliver the diagnosis and stop.** Report: symptom → evidence chain (queries and results per hop) → root cause → origin class → blast radius → recommended remediation and its risk. Hand off to `responding-to-data-incidents` for the fix, or to the human where the call is theirs (rollback of shipped numbers, severity, comms).

## Standards

- **Every hop has evidence.** A diagnosis that says "probably the connector" without a sync log or a row-count query is a hypothesis, not a finding — label it as one.
- **"Data wrong" vs "check wrong" is always stated explicitly.** The remediation differs completely; ambiguity here is how correct tests get deleted.
- **Do not fix while diagnosing.** No edits, no reruns that mutate state, no "quick" threshold bumps mid-investigation. Read-only until the diagnosis is delivered.
- **Known fixtures are recognized, not reported.** Deliberately failing tests, sandbox models, and disabled checks (look for TODO markers, `do_not_run` conventions, chronically-red-since-creation history) are excluded from findings with a note — flagging them as incidents destroys trust in every other finding.
- **Cost discipline.** Bisect with aggregates and counts, not `SELECT *`; sample before scanning; use time travel/snapshots instead of replaying pipelines.

## Common mistakes

| Mistake | Fix |
|---|---|
| Fixing the first plausible suspect | Verify each lineage hop; the visible breakage is usually a victim, not the cause |
| Trusting the alert's framing | Reproduce the symptom yourself first; monitors misfire and BI layers cache |
| Diagnosing from code reading alone | Code shows what *should* happen; only data shows what did |
| Skipping "boring" lineage hops | Fan-out and filter defects live in models nobody suspects |
| Declaring "test is wrong" because it's noisy | Prove data correctness against an independent authority first |
| Root-causing before sizing blast radius | Consumers are reading bad numbers while you investigate — surface the radius first |
| One diagnosis for two incidents | Concurrent symptoms may have distinct causes; bisect each separately |

## STOP if you're about to…

- recommend a fix without a verified causal chain from origin to symptom
- edit code, rerun a job, or change a threshold before the diagnosis is delivered
- report a chronically-red fixture or sandbox model as a new incident
- classify the origin as "test is wrong" without independent validation of the data
- close an investigation without stating blast radius and who is affected
