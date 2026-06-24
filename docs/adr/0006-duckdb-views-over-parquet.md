# ADR-0006: DuckDB views over Parquet, disposable and regenerable

- Status: Accepted
- Date: 2026-06-24
- Deciders: Jaime Casado

## Context

After extraction, users need a convenient way to query the local sample. DuckDB is the
chosen engine. The question is *how* DuckDB holds the data — and first, where it sits in
the architecture.

DuckDB is **downstream of the extraction port** (ADR-0001): it consumes the masked Parquet
in `LocalDataset`. It is the **local access layer**, fully decoupled from the engine — any
future adapter lands Parquet and the same DuckDB layer sits on top unchanged.

## Decision

- **One view per dataset** over `read_parquet(...)`.
- `.duckdb` stores **view definitions only** — no data. It is **disposable and
  regenerable** from the manifest + Parquet; delete it and `ldt` rebuilds it.
- **Parquet is the single source of truth.**
- DuckDB is explicitly **outside the port** — an engine-agnostic access layer.
- **No native-table materialization.**

## Consequences

### Positive
- Zero data duplication; `.duckdb` introduces no new (possibly unmasked) materialization —
  the masked Parquet remains the only copy, reinforcing the security model.
- Rebuildable convenience layer; nothing important is lost if `.duckdb` is deleted.

### Negative / costs
- Queries re-read Parquet each time rather than hitting pre-loaded tables. Negligible at
  sample scale, where DuckDB-on-Parquet is already fast.

## Alternatives considered
- **`COPY INTO` native DuckDB tables**: rejected — duplicates the data, creates a second
  source of truth that can drift from the manifest-referenced Parquet, bloats `.duckdb`,
  and turns it into a new data artifact outside the governed-Parquet story.

## Related
- ADR-0001 (the port DuckDB consumes), ADR-0005 (manifest used to regenerate views).
