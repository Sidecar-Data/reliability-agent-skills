---
name: authoring-data-quality-tests
description: Builds and maintains a data-quality safety net worth trusting — adding schema tests (not_null, unique, accepted_values, relationships) where they carry signal, setting source-freshness SLAs from observed arrival patterns, auditing monitor health, and tuning noisy alerts so that red means act. Use when adding test coverage to models or columns, recommending tests from coverage gaps, configuring freshness thresholds and alerting, auditing monitors for ownership and chronic breaches, or recalibrating alert fatigue. NOT for singular business-rule tests written as part of building a model (Builder persona), investigating why a test is currently failing (investigating-data-incidents), remediating the failure (responding-to-data-incidents), or defining enforced data contracts (Custodian persona).
metadata:
  persona: reliability
  author: sidecar
---

# Authoring Data Quality Tests

**Core principle: every red must mean act.** A test is a pager, not a checkbox — each one asserts something a human will respond to when it breaks. Coverage that nobody answers trains the team to ignore the color red, which is worse than no coverage: it silences the tests that matter. Signal per test beats tests per model.

## When to use

- Add schema tests to under-covered models and columns (primary keys first)
- Turn a coverage audit or recurring incident into a prioritized test plan
- Declare or correct source-freshness SLAs and their alerting
- Audit monitor/test health: no owner, chronically red, config drift
- Recalibrate noisy alerts that the team has learned to ignore

**Do NOT use for:** encoding a business rule as a singular test while building the model — tests written alongside new models belong to Builder's flow; diagnosing a currently-firing test (`investigating-data-incidents`); the in-the-moment decision to suppress or route a firing alert (`triaging-and-routing-alerts` — triage suppresses once with a rationale; this skill is where a *repeatedly* suppressed alert comes for permanent recalibration); enforced model contracts with schema guarantees (Custodian). If tuning reveals the *data* is wrong rather than the threshold, stop tuning — that's an incident, route to `investigating-data-incidents`.

## Step 0 — Gather context before writing any test

From **Sidecar platform context** (fall back to dbt artifacts + warehouse queries):

| Question | Where to look |
|---|---|
| What's covered today, and what passes? | Test definitions + run history per model — a coverage map, split into covered / partial / bare |
| What matters most? | Usage + exposures: query activity, dashboard popularity, downstream fan-out |
| What are the project's testing conventions? | Sibling models: where tests live (YAML layout), severity usage, custom generic tests already available |
| When does data actually arrive? | Observed load timestamps per source over ≥2 weeks — the empirical basis for freshness SLAs |
| Which failures are intentional? | Chronically-red-since-creation tests, TODO/disabled markers, sandbox/fixture folders |

From **the human**:
- Who responds when this test goes red? (No plausible responder → the test should be `warn` or shouldn't exist)
- Which SLAs are commitments to consumers vs aspirations? ("Finance needs it by 8am" vs "daily-ish")

Ask only what changes the test plan; state assumptions for the rest and proceed. Unknown responder → default the test to `warn` and flag the ownership gap in the PR description rather than blocking on the question.

## Workflow

1. **Map coverage against importance.** Cross the coverage map with usage: heavily-consumed bare models are the priority; sandbox projects and fixtures are explicitly out of scope. Deliverable: a ranked gap list with a one-line business-impact justification each — not "N models lack tests."
2. **Identify intentional fixtures before proposing anything.** Deliberately failing tests, disabled checks, `do_not_run`-style conventions: list them as exclusions with evidence. A test plan that "fixes" a fixture is wrong everywhere else too, as far as the reader trusts it.
3. **Author schema tests where the signal is:**
   - **Primary keys first**: `unique` + `not_null` on every staging and mart PK — the cheapest, highest-signal tests that exist. Match sibling-model YAML conventions exactly (file layout, `tests:` vs `data_tests:` key, inline vs block style).
   - **Grain guards** where joins change grain: composite-key uniqueness on intermediates.
   - `accepted_values` on **stable enums only** (status/type columns a code change controls) — never on organically growing sets like country or product lists.
   - `relationships` where orphaned foreign keys would corrupt downstream joins.
   - Do **not** spray `not_null` across every column: nullable-by-nature columns produce permanent noise.
4. **Set freshness SLAs from observation, not aspiration.** Query the actual arrival pattern; set `warn_after` just outside the observed p95 gap and `error_after` where consumers are genuinely harmed. Account for weekends/batch cadence (a Monday-morning alert on a source that never loads Sundays is self-inflicted noise). Record the SLA's basis in the YAML description.
5. **Tune the noisy estate.** For each chronic offender, decide deliberately: threshold wrong → recalibrate from the observed distribution; test asserts something nobody responds to → downgrade to `warn` or delete *with a stated rationale*; data genuinely bad → that's an incident, route it. Silence is chosen per test, never by blanket-disabling.
6. **Assign severity by response contract.** `error` = someone acts today (block/page); `warn` = reviewed on a cadence (drift, soft expectations). If you can't name the responder, it isn't an `error`.
7. **Ship through the delivery contract.** YAML-only diffs conforming to project conventions; description notes for anything non-obvious (why this threshold, who responds). Verify the touched files parse (`dbt parse`) and the new tests run against current data — a new test that's red on arrival is either a real finding (report it, don't merge it silently) or a bad test (fix it).

## Standards

- **Every `error`-severity test names a plausible responder** — in the description, an ownership tag, or the project's alert routing.
- **Thresholds come from queried data**, never from defaults or guesses; the evidence is recorded with the test.
- **Prioritization is by consumption**, not alphabetically or by folder: a bare model feeding ten dashboards outranks a covered one feeding none.
- **Deleting a test is legitimate maintenance** when it carries no signal — but always with a stated rationale, never silently, and never to unblock a run (that's the responding skill's hard rule too).
- **Additions match house style.** Test YAML layout, naming, and severity conventions follow the sibling models; a correct test in a foreign style is a review burden.

## Common mistakes

| Mistake | Fix |
|---|---|
| `not_null` sprayed on every column | PKs, grain guards, and a few critical rules; nullable-by-nature columns stay untested |
| Freshness SLA set to a round number | Query the observed arrival distribution; set warn/error from p95 + harm point |
| `accepted_values` on a growing set | Stable, code-controlled enums only |
| Everything at `error` severity | No nameable responder → `warn` or nothing |
| "Fixing" intentional fixtures | Identify and exclude them first, with evidence |
| Coverage measured in test counts | Measure in covered *risk*: PKs, grains, contracts on what's actually consumed |
| Shipping a new test that's red on arrival | Red on arrival = a finding to report or a bad test to fix — never merge-and-ignore |
| Tuning a threshold to silence a real data problem | If the data is wrong, it's an incident — route to investigation |

## STOP if you're about to…

- add an `error`-severity test with no plausible responder
- set a freshness SLA without having queried the actual arrival pattern
- propose tests for a known fixture, sandbox, or deliberately failing model
- disable or delete a failing test without a stated rationale (or to unblock a run)
- ship a test suite whose new tests you haven't executed against current data
