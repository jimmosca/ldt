# ADR-0004: Cost guardrail — `where`/`limit` floor + bytes-scanned cutoff + prune warning

- Status: Accepted
- Date: 2026-06-24
- Deciders: Jaime Casado

## Context

The original invariant was "every dataset must have a `where` or a `limit`." But Athena
bills on **bytes scanned**, and:

- A `WHERE` on a **non-partition** column scans the whole table, then filters — no saving.
- `LIMIT` does **not** reduce scan; Trino reads the data, then truncates rows.

So `where`-OR-`limit` only stops the laziest mistake (a bare unbounded extract). It is a
**friction floor, not a cost ceiling.** An ADR that claims otherwise would mislead.

## Decision

1. **Mandatory floor:** every dataset must declare a `where` or a `limit`; fail validation
   if neither is present. (Stated explicitly as friction, not a ceiling.)
2. **Real ceiling:** a **bytes-scanned cutoff** — Athena workgroup
   `BytesScannedCutoffPerQuery` and/or a per-run config cap — so a query *aborts* past N
   bytes.
3. **Prune warning:** at `plan` time, warn when the `where` touches **no Glue partition
   key** ("this filter won't prune; you'll scan the full table"). Uses Glue partition
   metadata only — no dry-run query, so `plan` stays AWS-cost-free.
4. **Visibility:** surface estimated/actual **bytes scanned** in `plan` and record actual
   bytes scanned in the manifest (→ ADR-0005, adapter block).

## Consequences

### Positive
- Honest cost story: a real abort-level ceiling plus pre-flight visibility.
- `plan` remains side-effect-free and free of charge.

### Negative / costs
- The partition-prune warning is best-effort (depends on Glue partition metadata being
  present and the predicate being statically analyzable).

## Alternatives considered
- **`where`/`limit` alone as the guardrail**: rejected — a false sense of safety; neither
  reliably reduces bytes scanned.

## Related
- ADR-0002 (where the SQL is built), ADR-0005 (bytes scanned recorded in adapter block).
