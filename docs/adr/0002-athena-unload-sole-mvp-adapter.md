# ADR-0002: Athena `UNLOAD` is the sole MVP extraction adapter

- Status: Accepted
- Date: 2026-06-24
- Deciders: Jaime Casado

## Context

ADR-0001 defines a coarse, engine-agnostic extraction port. For the MVP we need exactly
one adapter behind it. Athena is attractive because it **abstracts Delta vs Iceberg**
(Athena resolves the physical format via Glue), it **respects Glue + Lake Formation**
permissions, and `UNLOAD` writes **Parquet directly** to a location we choose.

## Decision

The only MVP adapter is `AthenaUnloadBackend`. It:

1. Builds an `UNLOAD(SELECT <projection, with masking> FROM <db.table> WHERE … ORDER BY …
   LIMIT …) TO 's3://<staging>/users/<identity>/<project>/<run_id>/<dataset>/' WITH
   (format='PARQUET', compression='SNAPPY')`.
2. Runs the query asynchronously and polls to completion.
3. Syncs the resulting Parquet from the staging prefix to local disk.

**Out of scope, explicitly:** local Delta/Iceberg readers and any direct read of
productive S3 paths. This is *not* an interface limitation — the port could in principle
host such an adapter — but no **governed local-read** story exists, and such an adapter
would defeat the security model of ADR-0001.

## Consequences

### Positive
- Format-agnosticism for free; the adapter never knows or cares about Delta vs Iceberg.
- Governance is inherited from Athena + Glue + Lake Formation.

### Negative / costs
- Bound to Athena availability, latency, and per-query cost.
- Operational note: an `UNLOAD` target prefix must be **empty**, or Athena errors — the
  adapter must ensure a clean per-run prefix (the `<run_id>` segment makes this natural).

## Alternatives considered
- **Plain Athena query** (CSV results in the workgroup output location): rejected — single
  gzipped CSV, less efficient, wrong artifact; `UNLOAD` yields partitioned Parquet directly.
- **Local Spark**: rejected — heavy, and an ungoverned direct-S3 read path.
- **`pyiceberg` / `deltalake` local readers**: rejected on governance (see ADR-0001).

## Related
- ADR-0001 (the port this adapter implements), ADR-0007 (IAM the adapter needs).
