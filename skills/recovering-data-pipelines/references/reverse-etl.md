# Recovery playbook — Reverse ETL (Hightouch, Census)

*Last reviewed: 2026-07. Destination behaviors are per-connector — verify against the destination's doc (Salesforce vs HubSpot vs ad platforms differ wildly).*

## Reading state

Reverse ETL failures have **two sides**; read both before acting:

- **Source side**: did the model/query the sync reads change (schema drift, missing PK, NULL keys, grain change)? A dbt refactor upstream is the most common "reverse ETL incident."
- **Destination side**: the per-record reject log — which rows failed and with what destination error (validation, required field, rate limit, permission, duplicate rule). Aggregate the reject reasons before anything: 10,000 rejects with one cause is one bug; 40 with 12 causes is data quality.
- **Sync mode and identity**: upsert vs update-only vs append, and the match key. This decides both the blast radius of the failure and the safety of a resync.
- **What actually landed**: the destination is a *production business system* — check what the partial sync already wrote (records created without required fields, half-updated audiences) — the failure may have left the destination in a worse state than "missing rows."

## Recovery verbs, in order of preference

1. **Fix the cause, reprocess the rejects** — the workhorse: correct the source-side issue (or the field mapping), then use the tool's retry-failed-records verb for exactly the rejected batch. Never full-resync to fix a reject batch.
2. **Resync after source repair** — when the upstream model was wrong and has been fixed (normal delivery flow): let the next sync diff-and-update. Verify the tool's diffing baseline is intact — some tools track state; a broken state store turns the next sync into a full push.
3. **Scoped full resync** — when sync-state is corrupted: confirm the destination-side effect first (an upsert full-resync is mostly safe; an *append*-mode full resync duplicates everything; update-only may mass-touch records and fire the destination's automations).
4. **Destination-side cleanup** — records the failed sync created wrongly (missing fields, wrong audience membership): fixing them may mean syncing a correction, or manual/bulk operations in the destination — **always approval-gated**, since this edits a production CRM/ad platform directly.

## Gotchas

- **Rejects are often the destination telling the truth**: "email invalid," "required field missing" are *data quality findings* about the warehouse model — route the pattern to `authoring-data-quality-tests` (add the validation upstream) instead of retrying the same bad rows forever.
- **Automations fire on writes**: destination systems trigger workflows (emails to customers, ad-audience updates, lead assignment) on the records you touch. A careless resync can send real emails. Estimate what a resync will *touch*, not just what it will fix — this is the biggest blast-radius trap in the whole recovery domain.
- **Rate limits compound**: retrying a large batch into the same rate limit that caused the rejects produces the same rejects; chunk and pace per the destination's documented limits.
- **Match-key NULLs**: NULL or duplicated match keys upsert unpredictably (create-instead-of-update storms). Check key integrity in the source model before any retry.
- **Deletes rarely propagate**: rows removed from the source audience may not be removed downstream depending on mode — "recovered" sync with stale audience members still in the destination is a half-recovery; say so.

## Verify

- Reject count on the reprocess ≈ 0, and the *reasons* for any remainder are new (not the original cause re-failing).
- Destination spot-check: sampled records show the correct field values and audience membership — via the destination's UI/API, not just the sync tool's green check.
- No unintended automation storm (check the destination's activity/audit log for the sync window).
- Source-side data-quality findings from the reject analysis filed to `authoring-data-quality-tests`.
