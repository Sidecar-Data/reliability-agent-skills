---
name: monitoring-feature-freshness-and-drift
description: Watches production ML feature tables for the two failure modes that corrupt predictions silently — staleness (features not updating on schedule) and drift (feature distributions moving away from what the model was trained on) — establishing baselines, choosing drift metrics and thresholds, wiring monitors, and routing breaches into the incident flow. Use when setting up monitoring for feature tables or a feature-store layer, when a model's predictions degrade and stale or drifted inputs are suspected, when defining freshness SLAs for features consumed by online or batch scoring, or when reviewing existing feature monitors for coverage gaps. NOT for building the feature tables themselves (Builder persona), diagnosing a specific firing alert (investigating-data-incidents), model-quality monitoring beyond input features (accuracy, calibration — the ML team's stack), or generic table-level tests on non-feature data (authoring-data-quality-tests).
metadata:
  persona: reliability
  author: sidecar
---

# Monitoring Feature Freshness and Drift

**Core principle: a model fails at the speed of its stalest feature.** Feature pipelines fail differently from analytics pipelines: a stale dashboard gets noticed by a human, a stale feature gets consumed by a model that keeps predicting — confidently, wrongly, silently. Feature monitoring exists to make invisible degradation visible *before* the business metric it feeds moves.

## When to use

- Set up freshness and drift monitoring for feature tables or a feature-store layer
- Define feature-level SLAs matched to how the model consumes them (online scoring vs nightly batch)
- Predictions degraded and stale/drifted inputs are suspected — establish which feature moved, when
- Audit existing feature monitors: what's watched, what's not, what's noise

**Do NOT use for:** building or fixing the feature pipelines (Builder's territory; a broken feature *pipeline* is a normal incident — `investigating-data-incidents` then `recovering-data-pipelines`); monitoring model outputs (accuracy, calibration, prediction drift — that's the ML team's model-observability stack; coordinate, don't duplicate); generic schema tests on non-feature tables (`authoring-data-quality-tests` — this skill *extends* it with distribution-level checks that ordinary tables don't need).

## Step 0 — Gather context before wiring any monitor

From **Sidecar platform context**:

| Question | Where to look |
|---|---|
| Which tables are features, and which models consume them? | Catalog tags/naming (`feature_`, feature-store schemas), lineage to scoring jobs and reverse-ETL to serving |
| How does each model consume? (online per-request, nightly batch, training-only) | Downstream lineage + job schedules — consumption cadence sets the SLA |
| What do the features look like today? | Profile every monitored column: type, null rate, cardinality, distribution — the baseline candidates |
| What already fires? | Existing tests/monitors on these tables — extend, don't duplicate |

From **the human / ML team** (this context lives outside the warehouse — ask):
- Which features actually matter to the model? (top-importance features get tight monitors; the tail gets coarse ones — monitoring 400 features equally is noise manufacturing)
- What window was the model trained on? (the training window is the drift baseline, not "last month")
- Who owns model retraining, and what drift magnitude would trigger it? (the threshold is a retraining decision, not a statistics decision)

## Workflow

1. **Inventory and rank.** Feature tables from catalog + lineage; features ranked by model importance (from the ML team) and consumption criticality. The monitor budget goes to the head of that list.
2. **Freshness: SLA per consumption pattern.** Online-scored features get SLAs derived from the feature pipeline's schedule with tight margins; batch-scored features need freshness *relative to the scoring job* — "updated before tonight's scoring run" is the real contract, not "updated daily." Wire as source-freshness checks / staleness monitors per `authoring-data-quality-tests` mechanics (observed arrival patterns, warn at p95, error at harm).
3. **Baseline for drift from the training window.** Snapshot each ranked feature's distribution over the model's training period: numeric → quantiles/mean/variance/null rate; categorical → frequency table + cardinality; store the baseline where the monitor can query it. Baselines are versioned artifacts — retraining refreshes them.
4. **Choose drift checks that fit the feature, thresholds that fit retraining.** Population-stability / KS-style distribution comparison on the important numerics; frequency-shift and new-category detection on categoricals; null-rate and cardinality deltas everywhere (they catch pipeline bugs masquerading as drift). Set thresholds with the ML team against "would this move retrain us?" — flagging drift nobody would act on is the noisy-alert anti-pattern with extra math.
5. **Distinguish drift from breakage in the monitor design.** A feature whose null rate jumped to 90% didn't drift — its pipeline broke. Route the two differently at the monitor level: staleness/null/cardinality breaches → normal incident flow (`triaging-and-routing-alerts`); genuine distribution drift → the retraining owner, with the comparison evidence. Conflating them pages the wrong people for both.
6. **Wire into the incident flow, not a parallel one.** Feature monitor breaches enter the same triage → investigate → respond loop as everything else; a stale feature table is contained by pausing/flagging the *scoring* consumer (with the ML team) the way a stale mart is contained by warning dashboard owners.
7. **Review on retrain.** Every model retrain invalidates baselines and possibly thresholds; refreshing them is part of the retrain checklist. A drift monitor comparing against a two-generations-old training window is itself drifted.

## Standards

- **Baseline = training window, versioned, refreshed on retrain.** Never "trailing 30 days" — a slowly drifting feature drags a trailing baseline with it and the monitor never fires.
- **Monitor by importance, not by column count.** Tight monitors on the features that drive predictions; coarse pipeline-health checks (freshness, null rate, volume) on the rest.
- **Every threshold has a named consequence** — "breach ⇒ retraining evaluation by ⟨owner⟩" or "breach ⇒ incident" — matching authoring's every-red-means-act rule.
- **Pipeline breakage is never reported as drift.** Null-rate spikes, cardinality collapses, and staleness are incidents; drift reports must survive the question "or is the pipeline just broken?"
- **Coordinate the boundary with the ML team.** Inputs (this skill) vs outputs (their stack) — agree who watches what, so degradation can't fall between two monitoring systems.

## Common mistakes

| Mistake | Fix |
|---|---|
| Monitoring all features equally | Rank by model importance; budget monitors to the head |
| Trailing-window drift baselines | Baseline on the training window; refresh on retrain |
| Drift thresholds from statistics alone | Set with the ML team against the retraining decision |
| Reporting a broken pipeline as "drift" | Null/cardinality/staleness checks fire first and route as incidents |
| Freshness SLA of "daily" for batch-scored features | SLA relative to the scoring job's schedule |
| Standalone feature-monitoring silo | Breaches enter the standard triage → investigate → respond flow |
| Baselines never versioned | Retrain invalidates baselines; stale baselines alert on the wrong world |
| Duplicating the ML team's output monitoring | Inputs here, outputs theirs — agree the boundary explicitly |

## STOP if you're about to…

- set a drift threshold without the retraining owner agreeing what a breach triggers
- baseline drift on a trailing window instead of the training window
- report distribution drift without having ruled out pipeline breakage
- wire feature alerts into a channel outside the standard triage flow
- monitor a feature set without knowing which features the model actually weights
