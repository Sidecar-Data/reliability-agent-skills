# Reliability Persona — Initial Skill Set (v0)

The Reliability agent is the data team's incident responder — it owns failure response and data
quality (**28 of the 120 workflows** in `data_team_workflows.csv`, including Data Quality &
Incident Response nearly wholesale). Like its sibling
[`builder-agent-skills`](https://github.com/Sidecar-Data/builder-agent-skills), skills are chosen
for bang-for-buck: seven skills covering all 28 Reliability workflows.

## The portfolio

| Skill | One-liner | Workflows covered (IDs) |
|---|---|---|
| [`triaging-and-routing-alerts`](skills/triaging-and-routing-alerts/SKILL.md) | The event front door: normalize, dedupe, size severity from blast radius, find the owner, disposition in minutes | 38, 40 |
| [`investigating-data-incidents`](skills/investigating-data-incidents/SKILL.md) | Symptom → time bisection → lineage walk → root cause → origin classification (source vs ETL vs transform vs wrong check) | 32, 33, 34, 35, 36, 37, 45, 109 |
| [`responding-to-data-incidents`](skills/responding-to-data-incidents/SKILL.md) | Incident command: contain, choose the smallest remediation, verify restoration, postmortem, runbook | 10, 24, 41, 42, 46, 104 |
| [`recovering-data-pipelines`](skills/recovering-data-pipelines/SKILL.md) | Tool-level recovery mechanics with **per-tool playbooks**: dbt, orchestrators, connectors, warehouse, reverse ETL | 4, 5, 7, 116 |
| [`communicating-data-operations`](skills/communicating-data-operations/SKILL.md) | Incident notices, cadenced status updates, verified all-clears, planned-maintenance announcements, postmortem distribution | 39, 110 |
| [`authoring-data-quality-tests`](skills/authoring-data-quality-tests/SKILL.md) | Schema tests, freshness SLAs from observed data, monitor health, alert-noise recalibration | 28, 29, 31, 43, 44 |
| [`monitoring-feature-freshness-and-drift`](skills/monitoring-feature-freshness-and-drift/SKILL.md) | ML feature tables: staleness SLAs and training-window drift baselines, routed into the standard incident flow | 120 |

**Coverage: 28 of 28 Reliability workflows.** The skills compose into one loop:
*intake (triage) → diagnose (investigate) → restore (respond, with recover for the tool mechanics)
→ inform (communicate, throughout) → prevent (author tests; monitor features)*. Triage is the
entry point for every event; investigation always precedes remediation; response delegates
tool mechanics to recovery and all stakeholder-facing messages to communication; anything that
reveals a coverage gap terminates in test authoring.

## Tool playbooks & the flywheel

`recovering-data-pipelines/references/` ships per-tool recovery playbooks — [dbt](skills/recovering-data-pipelines/references/dbt.md),
[orchestrators](skills/recovering-data-pipelines/references/orchestrators.md) (Airflow/Dagster/Prefect),
[connectors](skills/recovering-data-pipelines/references/connectors.md) (Fivetran/Airbyte/Stitch),
[warehouse](skills/recovering-data-pipelines/references/warehouse.md) (Snowflake/BigQuery),
[reverse ETL](skills/recovering-data-pipelines/references/reverse-etl.md) (Hightouch/Census) —
each covering state-reading, sanctioned recovery verbs in preference order, gotchas, and
verification. Like the Builder's source playbooks, they are priors, not truth: volatile details
are verified against the installed version, each playbook carries a "Last reviewed" date, and
recoveries that reveal gaps draft the playbook addition as part of the change (**the flywheel**).

## Repository layout

```
skills-manifest.yaml        # profiles.reliability — the ingest entry point
skills/
  <slug>/SKILL.md           # one bundle per skill, metadata.persona: reliability
  <slug>/references/        # conditional detail, loaded per situation
```

Consumed by service-platform's `INGEST_AGENT_SKILLS` admin task
(`{"agent_identifiers": ["RELIABILITY"]}`), which clones this repo at the tag pinned by
`RELIABILITY_AGENT_SKILLS_REF` and uploads the `reliability` profile bundles to Anthropic.

**Versioning: ingest pins tags, never branches.** Cut a `vX.Y.Z` tag when the bundles are ready;
do not point deployments at `main`. No tag exists yet — the first one (`v0.0.1`) ships with the
initial bundles.

## Deliberately deferred (v2+)

- **BI-tool recovery playbooks** (Looker/Tableau/Power BI cache and extract failures): today the
  BI layer is where symptoms *surface* (handled in investigation); tool-level BI recovery earns a
  playbook when demand shows up.
- **Chat/ticket integration mechanics** (Slack/Jira/Linear API specifics for comms and incident
  records): the comms skill defines *what* to send and *where* decisions live; per-tool mechanics
  arrive with the platform's MCP/integration configuration.
- **On-call schedule management**: triage escalates along a ladder it's told about; owning the
  rotation itself (PagerDuty/Opsgenie) is out of scope.

## Authoring conventions (shared with builder-agent-skills)

- **Frontmatter:** `name` (gerund, matches directory), `description` (capability statement +
  "Use when…" + negative triggers routing to sibling skills and other personae),
  `metadata.persona: reliability`.
- **Body:** Core principle → When to use / do NOT use → Context to gather first → numbered
  workflow → standards → Common Mistakes table → STOP tripwires.
- **Thin SKILL.md, conditional references** — detail loads per situation (`references/`).
- **Evergreen method, runtime facts.** Skills contain durable *method*; customer-specific facts
  (schemas, monitors, owners, lineage) come from Sidecar platform context at runtime. Never bake
  a customer's setup into skill prose.
- **Diagnose-first.** Reliability skills never prescribe a fix before the workflow has established
  root cause and classified whether the data or the test/monitor is wrong.

## Persona boundaries

Route away, don't absorb: designing or building new models, connectors, or DAGs → Builder
(`designing-data-models`, `onboarding-data-sources`); reviewing another team's breaking change or
contract enforcement → Custodian; cost/performance tuning (warehouse sizing, retry/timeout tuning
as a project, resync spend) → Optimization; metric definitions, discovery questions, and
steady-state trust answers → Concierge.
