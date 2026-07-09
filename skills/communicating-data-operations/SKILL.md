---
name: communicating-data-operations
description: Writes the operational communications around data incidents and maintenance — initial incident notices, cadenced status updates, the verified all-clear, planned-maintenance announcements, and postmortem distribution — audience-scoped so consumers hear what affects them before they discover it themselves. Use when an incident is opened and stakeholders need notice, when an ongoing incident needs a status update, when recovery is verified and consumers need the all-clear, when a maintenance window or planned outage must be announced, or when a postmortem is ready to share. NOT for deciding severity or routing (triaging-and-routing-alerts), diagnosing (investigating-data-incidents), executing fixes or writing the postmortem's technical content (responding-to-data-incidents), or answering general data-discovery and trust questions outside an incident (Concierge persona).
metadata:
  persona: reliability
  author: sidecar
---

# Communicating Data Operations

**Core principle: consumers hear it from you first.** A stakeholder who discovers bad data on their own dashboard has lost trust twice — once in the data, once in the team that knew and didn't say. Operational comms are part of the remediation, not an afterthought: what is affected, what it means for the reader, what's being done, when they'll hear more. Never faster than the facts.

## When to use

- An incident was just opened → initial notice to affected consumers
- An incident is ongoing → cadenced status updates ("still investigating" *is* an update)
- Recovery is verified → the all-clear, with what was wrong and for how long
- A maintenance window, migration, or planned outage needs announcing
- A completed postmortem needs distribution to stakeholders

**Do NOT use for:** deciding *whether* something is an incident or how severe (`triaging-and-routing-alerts`); establishing the facts being communicated (`investigating-data-incidents` supplies the diagnosis, `responding-to-data-incidents` supplies remediation state and the postmortem's substance — this skill shapes and distributes, it never invents); steady-state "how fresh is this table?" questions with no incident behind them (Concierge).

## Step 0 — Gather context before drafting

- **The facts, from their owners:** affected assets and blast radius (from the diagnosis), remediation state and ETA-confidence (from the response), severity and incident record (from triage). No verified fact → the update says "under investigation," not a guess.
- From **Sidecar platform context**: who actually consumes the affected assets — exposures, dashboard owners, top queriers. The notification list is derived from usage, not from a static "data-consumers" alias (though org broadcast channels supplement it).
- From **the org** (learn once, reuse): where each audience is reached (Slack channels, email lists, status page, Jira/Linear), and any incident-comms template the org already uses — conform to it.

## Workflow

1. **Scope the audience from lineage, not habit.** Directly affected consumers (their dashboard/model/export is wrong or stale) get direct, specific notice. Broad channels get the summary. Uninvolved teams get nothing — over-broadcasting is how notices stop being read.
2. **Lead with reader impact, not mechanism.** First sentence: what the reader experiences and since when — "Revenue dashboards have been stale since 06:00 UTC; numbers shown are yesterday's." Mechanism ("the Fivetrans sync stalled") goes below, for those who care.
3. **State knowns, unknowns, and the next update time.** Every incident message carries three things: what is established (verified facts only), what is not yet known, and when the next update comes. A named next-update time is what stops the DM flood.
4. **Cadence by severity.** High-severity: short-interval updates even when the update is "no change." Lower severity: on state change. Never let a promised update time lapse silently — "still working, next update at 15:00" costs one line.
5. **All-clear only on verification.** The recovery notice ships when reconciliation evidence exists (from `responding-to-data-incidents`), and it says: what was wrong, the affected window, what was done, and whether consumers must act (re-pull an export, refresh a dashboard, disregard numbers seen between T1–T2). An all-clear that later reverses costs more trust than the incident itself.
6. **Planned maintenance: announce → remind → confirm.** The announcement names affected assets (from lineage — not "some pipelines"), the window in consumer-relevant time, what consumers should do (nothing / avoid relying on X / expect staleness until Y), a reminder near the window, and a confirmation (or overrun notice) at the end. Route the window to triage's context so its alert echoes are suppressed with a pointer, not investigated.
7. **Distribute the postmortem.** Take the completed blameless postmortem (authored in `responding-to-data-incidents`), produce the stakeholder-facing summary — impact, root cause in plain language, what changes to prevent recurrence — and post it where the incident was announced, closing the loop with the same audience that saw it open.

## Standards

- **Never ahead of the facts.** No root-cause claims before the diagnosis confirms one; no ETA without the responder's confidence; no all-clear without verification evidence. "We don't know yet" is a professional sentence.
- **Timestamps with timezones, windows with both ends.** "Since 06:00 UTC," "between 02:00–09:30 UTC" — never "this morning" or "recently."
- **One incident, one thread.** Updates land in the same thread/ticket as the initial notice so latecomers see the full history; cross-posts link to the canonical thread rather than duplicating it.
- **Blameless in public, always.** External comms never name the person or team that caused the incident; systems and safeguards fail, not people.
- **Outbound messages to external channels are approval-gated.** Draft, show the human, send on approval — per the persona's approval policy. Drafting is autonomous; publishing is not.

## Common mistakes

| Mistake | Fix |
|---|---|
| First sentence explains the pipeline | Lead with what the reader experiences; mechanism below |
| Waiting for full diagnosis before any notice | Notify on incident open with knowns/unknowns; update as facts land |
| Promising an ETA to seem in control | State the next-update time instead; ETAs come from responders with confidence |
| All-clear on "the job went green" | All-clear requires reconciliation evidence; a reversed all-clear is the costliest message there is |
| Broadcasting every incident to everyone | Audience from lineage and usage; over-broadcast trains people to skim |
| Maintenance notice says "some data may be delayed" | Name the assets, the window, and the consumer action |
| Letting a promised update lapse | "No change, next update at T" costs one line and preserves trust |
| New thread per update | One canonical thread per incident |

## STOP if you're about to…

- send an external message without human approval
- state a root cause, ETA, or all-clear that the diagnosis/response evidence doesn't back
- announce maintenance without naming affected assets and the consumer action
- broadcast an incident to audiences whose assets aren't in the blast radius
- close an incident thread without an all-clear that states the affected window
