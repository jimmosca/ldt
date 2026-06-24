# Architecture Decision Records

Decisions are immutable once Accepted; supersede with a new ADR rather than editing.

| # | Decision | Status |
|---|----------|--------|
| [0001](0001-extraction-behind-coarse-engine-agnostic-port.md) | Extraction behind a coarse, engine-agnostic port | Accepted |
| [0002](0002-athena-unload-sole-mvp-adapter.md) | Athena `UNLOAD` is the sole MVP extraction adapter | Accepted |
| [0003](0003-fail-closed-masking-closed-type-set.md) | Fail-closed masking with a closed, table-driven type set | Accepted |
| [0004](0004-cost-guardrail-where-limit-and-bytes-scanned.md) | Cost guardrail: `where`/`limit` floor + bytes-scanned cutoff + prune warning | Accepted |
| [0005](0005-two-layer-manifest.md) | Two-layer manifest: engine-neutral core + adapter block | Accepted |
| [0006](0006-duckdb-views-over-parquet.md) | DuckDB views over Parquet, disposable/regenerable | Accepted |
| [0007](0007-least-privilege-iam-documented-not-required.md) | Least-privilege IAM: documented & role-recommended, not required | Accepted |
