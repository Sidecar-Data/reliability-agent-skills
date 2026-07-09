# Recovery playbook — Managed connectors (Fivetran, Airbyte, Stitch)

*Last reviewed: 2026-07. Connector behaviors are per-source and per-version — verify in the vendor's connector doc for the specific source.*

## Reading state

- **Sync status and the actual error**: the connector's sync log for the failed sync — auth failure, source API change, rate limit, schema drift, destination write failure. Each maps to a different verb; "resync" fixes none of the first three.
- **Cursor position**: where the incremental cursor / replication key sits per table. This decides whether the next sync self-heals the gap or permanently skips it.
- **Schema-drift state**: pending schema changes the connector detected but hasn't applied (new/renamed/retyped columns, new tables) — many "failures" are actually the connector pausing for a human schema decision.
- **Soft-delete/history mode**: does the connector mark deletes (`_fivetran_deleted`-style flags) or hard-sync? Determines what a resync will do to history the warehouse currently holds.

## Recovery verbs, in order of preference

1. **Fix the cause, let the schedule self-heal** — auth: re-authorize (credential changes are approval-gated); rate limit: wait/raise limits; source API version change: connector update. For cursor-based syncs an unpaused connector usually catches up the gap on its own — check the cursor, don't reflexively resync.
2. **Approve/apply the schema change, then resume** — for drift-paused syncs: accept the new schema (or map it) through the connector's schema UI/API. Downstream (staging models) may need the corresponding dbt change — that's a normal change via delivery, coordinated with this recovery.
3. **Table-scoped resync** — re-extract only the affected tables when their state is wrong (cursor corrupted, partial load, drift mis-applied). This is the workhorse verb.
4. **Full connector resync** — last resort: re-extracts everything, can take days on large sources, hammers source API quotas, and (depending on destination mode) may drop-and-recreate warehouse tables mid-day. Approval-gated; announce a maintenance window (`communicating-data-operations`) if consumers will see truncated tables during the reload.

## Gotchas

- **Resync ≠ repair for history**: sources that only expose current state (most SaaS APIs) cannot restore deleted/changed history a broken period failed to capture — that history is gone unless snapshots caught it. Say so in the report; don't imply the resync recovered it.
- **Cursor reset semantics differ**: some connectors' "resync" resets cursors to zero (full re-pull), others re-window. Know which before triggering, or the "quick fix" becomes a multi-day reload.
- **Destination-side failures masquerade as connector failures**: permissions revoked on the warehouse target, or a dbt model reading a table mid-swap. Check the destination half of the log before touching the connector.
- **MAR/row-quota side effects**: a large resync can spike billed volume — flag expected cost on big resyncs (that's Optimization's domain to tune, but the flag is yours to raise).
- **Duplicate risk on at-least-once delivery**: connector retries can double-deliver into append-style destinations; dedupe in staging is the fix (Builder change), but the recovery report must note observed duplicates.

## Verify

- Sync completes green **and** the affected tables' row counts/max-updated-at reconcile vs the source (source-side counts or spot checks via the source API/UI).
- Cursor sits at/near now, not at zero and not stuck at the failure point.
- Downstream freshness checks recover on the next dbt run; staging models still parse (schema drift may have added/renamed columns).
- Next *scheduled* sync (not just the manual one) completes green.
