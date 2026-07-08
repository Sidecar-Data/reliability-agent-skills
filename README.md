# Reliability Persona — Initial Skill Set (v0)

The Reliability agent is the data team's incident responder — it owns failure response and data
quality (33 of the 120 workflows in `data_team_workflows.csv`, including Data Quality & Incident
Response nearly wholesale). Like its sibling [`builder-agent-skills`](https://github.com/Sidecar-Data/builder-agent-skills),
this is a deliberately small initial set: **3 skills chosen for bang-for-buck, not exhaustive coverage.**

## The portfolio (planned — bundles land next)

| Skill | One-liner | Workflows covered (IDs) |
|---|---|---|
| `investigating-data-incidents` | Symptom → lineage walk → root cause → origin classification (source vs ETL vs transform vs model) | 32, 33, 34, 35, 36, 37, 45, 109 |
| `responding-to-data-incidents` | Remediate with the smallest safe change: backfill, rerun, revert, quarantine — plus incident comms, postmortems, runbooks | 4, 5, 7, 10, 24, 38–42, 46, 104, 116 |
| `authoring-data-quality-tests` | Schema tests, freshness SLAs, monitor thresholds, and alert-noise tuning | 28, 29, 31, 43, 44 |

The three compose into one arc: *detect and diagnose → restore correctness → prevent recurrence.*
Diagnosis (`investigating-data-incidents`) always precedes remediation; remediation that reveals a
coverage gap terminates in `authoring-data-quality-tests`.

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

Route away, don't absorb: designing or building new models → Builder (`designing-data-models`);
reviewing another team's breaking change or contract enforcement → Custodian; cost/performance
tuning → Optimization; metric definitions and discovery questions → Concierge.
