# Billing statement retention and compaction

This document covers retention and compaction for the **per-charge audit log**
(`statements.rs`).  Period billing statements (`billing_statements.rs`,
`PeriodBillingStatement`) are stored separately and are **not** affected by
compaction — see `docs/billing_statements.md`.

This contract supports configurable retention and explicit compaction for billing statements.

## Retention model

- Detailed rows are append-only per subscription, keyed by sequence.
- `set_billing_retention(admin, keep_recent)` configures the default number of detailed rows to keep during compaction.
- `get_billing_retention()` returns current policy.
- Statements are not pruned on append. The first retention setting and each reduction compact existing histories; explicit compaction is needed to enforce the policy on later appends.

## Compaction flow

- **Explicit compaction**: `compact_billing_statements(admin, subscription_id, keep_recent_override)`
  - Admin-only operation
  - Prunes oldest detailed rows for one subscription (useful for per-run overrides or targeted cleanup)
- **Retention reduction**: lowering `keep_recent` with `set_billing_retention` immediately compacts all allocated subscription histories in the same transaction. Each affected history emits `billing_compacted`; pruned amounts remain available in its aggregate.
- Preserves high-level auditability in `BillingStatementAggregate`:
  - `pruned_count`
  - `total_amount`
  - `oldest_period_start`
  - `newest_period_end`

Read aggregate with `get_stmt_compacted_aggregate(subscription_id)`.

## Guarantees and limits

- Compaction never mutates remaining detailed rows.
- Remaining rows preserve sequence IDs and ordering.
- Pagination APIs continue to return stable ordering over active rows.
- Pruned detail is intentionally irreversible; only aggregate totals remain.

## Guidance

- Choose `keep_recent` to match frontend history window (for example, 12-24 periods).
- A retention reduction performs an immediate global cleanup. This pass scans subscription IDs and can be resource-intensive on contracts with many subscriptions; the config update and cleanup are atomic, so a failed cleanup leaves the old policy in place.
- Explicit compaction may be needed after additional statements are appended under the configured policy, because appends do not compact automatically.

## Security and operations

- Only the contract admin may call `set_billing_retention` and `compact_billing_statements` (see `require_admin_auth` in the vault). Holders of other roles cannot change retention or prune history.
- `keep_recent == 0` is allowed: a compaction run can remove **all** detailed rows for a subscription while cumulative pruned amounts and period bounds remain in `BillingStatementAggregate` (via `get_stmt_compacted_aggregate`).
- Each successful compaction emits `billing_compacted` with a summary that includes run totals (`pruned_count`, `kept_count`, `total_pruned_amount`) and the post-run aggregate (`aggregate_*` fields) so indexers can reconcile against `get_stmt_compacted_aggregate` without extra reads.
- Repeated compaction with the same effective threshold is a no-op (zero rows pruned) once the live row count is at or below the keep threshold—safe to run on a schedule.
- When tuning retention in production, prefer staged changes (set default policy, then compact high-traffic subscriptions with `keep_recent_override` before lowering the global default) so operators can validate exports and dashboards.
