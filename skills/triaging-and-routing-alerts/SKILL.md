---
name: triaging-and-routing-alerts
description: Runs the intake desk for data alerts — normalizing what fired, deduplicating against open incidents and known noise, sizing severity from blast radius, finding the owner, and deciding the disposition — investigate now, open an incident, route to the owner, or suppress with a stated rationale. Use when an alert fires (freshness, volume, test failure, monitor breach, job failure) and needs a fast routing decision, when multiple alerts may share one root cause, when an alert needs escalation because nobody acknowledged it, or when a real incident needs to be opened with an owner and severity. NOT for the actual diagnosis (investigating-data-incidents), the fix (responding-to-data-incidents), recalibrating chronically noisy thresholds (authoring-data-quality-tests), or writing the stakeholder updates once an incident is open (communicating-data-operations).
metadata:
  persona: reliability
  author: sidecar
---

# Triaging and Routing Alerts

**Core principle: every alert leaves triage with an owner and a decision.** An alert is a claim, not an incident — but an unrouted alert is worse than either: it trains the team that red things can be ignored. Triage is a minutes-scale activity; depth belongs to investigation. The output of triage is never "looked at it" — it is one of four dispositions, written down.

## When to use

- An alert fires and needs a disposition: freshness, volume, test failure, custom monitor, job failure, or an inbound "data looks broken" report
- Several alerts arrive together and may share one upstream cause
- A routed alert sat unacknowledged and needs escalation
- A confirmed problem needs a formal incident: severity, commander, tracking

**Do NOT use for:** root-causing (hand to `investigating-data-incidents` — triage decides *whether and to whom*, not *why*); fixing (`responding-to-data-incidents`); permanently fixing a noisy alert's threshold (`authoring-data-quality-tests` — triage may suppress once, recalibration is authoring work); drafting the consumer-facing comms (`communicating-data-operations`).

## Step 0 — Gather context (fast — this whole skill runs in minutes)

From **Sidecar platform context**:

| Question | Where to look |
|---|---|
| Is this alert already known? | Open incidents, recent alert history, this check's pass/fail record — chronically red since creation ⇒ known noise, not news |
| Do concurrent alerts share an ancestor? | Lineage: multiple firing assets with one common upstream ⇒ one incident, not N |
| How big is the blast radius? | Lineage + exposures + usage of the affected assets — severity comes from consumption, not from the alert's own severity field |
| Who owns this? | Ownership metadata, model/source `meta` owner tags, CODEOWNERS on the model path, the team that merged the last change |
| Is this expected? | Known fixtures, in-progress backfills, announced maintenance windows |

From **the human / the org**: the escalation ladder (who is on call, who is the fallback), and where incidents are tracked (Slack channel, Jira/Linear project) — learn once, reuse. Unknown ladder or tracker → still produce the disposition with the routing target left as a named gap ("route to owner of X — owner metadata missing, flagging") rather than stalling; the disposition decision never waits on org plumbing.

## Workflow

1. **Normalize.** One line: what fired, on which asset, when, and what the check asserts. Alerts arrive in tool-specific dialects; triage decisions are made on the normalized form.
2. **Dedupe before anything else.** Match against open incidents (same asset, same lineage subtree, same time window) and against the check's own history. Duplicate of an open incident → attach it there, done. Chronically red / known fixture / active maintenance window → suppress **with a written rationale and a pointer** (and if it's the third suppression of the same alert, route the alert itself to `authoring-data-quality-tests` as a noise finding).
3. **Cluster concurrent alerts.** Walk lineage upward from every currently-firing asset; alerts sharing a common failing ancestor are one incident named after the ancestor, not many named after the victims.
4. **Size severity from blast radius, in consumer terms.** Heavily-queried mart feeding executive dashboards ⇒ high. Sandbox model nobody queries ⇒ log-and-batch. The alert's own configured severity is an input, not the answer — an `error` on an unused table outranks nothing.
5. **Decide the disposition — exactly one:**
   - **Investigate now** — significant radius or ambiguous cause → hand to `investigating-data-incidents` immediately
   - **Open an incident** — user-facing impact or multi-team response needed: create the tracking item with severity, commander (a human — the agent never self-appoints as commander), affected assets, and the evidence so far; then trigger `communicating-data-operations` for the initial notice
   - **Route to owner** — clear single-owner issue below incident threshold: notify with the normalized summary and what response is expected by when
   - **Suppress with rationale** — known noise, fixture, or maintenance echo: written note, never silence
6. **Escalate on silence.** A routed alert has an acknowledgment window matched to its severity. Window passes → move up the ladder (owner → team → on-call), stating what was routed, when, and to whom. Escalation is a feature of the process, not an accusation.

## Standards

- **Minutes, not hours.** Anything requiring more than a few queries to disposition is by definition "investigate now" — stop triaging and route it.
- **Dedupe against *seen*, not just *open*.** An alert judged noise yesterday and suppressed is still noise today — but a third repeat is a signal about the check, not the data; route it to authoring.
- **Every suppression is written down** with rationale and evidence. Silent suppression is indistinguishable from a dropped alert.
- **Severity language is consumer impact** ("finance dashboards stale since 06:00"), never mechanism ("freshness check breached 3h threshold") — mechanism goes in the detail, not the headline.
- **The agent routes and recommends; humans command.** Opening an incident assigns a proposed severity and a proposed commander; both are human-confirmable calls.

## Common mistakes

| Mistake | Fix |
|---|---|
| Investigating during triage | Disposition in minutes; depth belongs to `investigating-data-incidents` |
| One incident per firing alert | Cluster by lineage ancestor first; victims are not incidents |
| Trusting the alert's configured severity | Size from blast radius and consumption |
| Suppressing silently | Rationale + pointer, always; repeated suppression → authoring finding |
| Routing to a team instead of a person | Ownership lookup names a responder; "the data team" is nobody |
| Treating maintenance-window echoes as incidents | Check announced windows and in-flight backfills before dispositioning |
| Escalating never (or instantly) | Ack windows matched to severity; escalate when they lapse, not before |

## STOP if you're about to…

- spend more than a few queries on a triage decision (→ route to investigation instead)
- suppress an alert without writing the rationale down
- open an incident without a proposed owner and severity
- disposition an alert while an overlapping open incident covers the same lineage subtree (→ attach, don't fork)
- let a routed high-severity alert pass its ack window without escalating
