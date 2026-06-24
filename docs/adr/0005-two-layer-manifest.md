# ADR-0005: Two-layer manifest — engine-neutral core + namespaced adapter block

- Status: Accepted
- Date: 2026-06-24
- Deciders: Jaime Casado

## Context

The manifest must let a reviewer **reproduce + audit** a sample. As the design settled, it
accumulated fields that are intensely Athena-specific — `query_execution_id`,
`bytes_scanned`, and the **generated `UNLOAD` SQL** (the single best human-readable proof
that masking was applied). This is the **same altitude problem** ADR-0001 solved for the
port: hard-coding Athena-shaped fields at the top level would make the manifest leak the
engine, and a future adapter would have to write `null`s into Athena-only fields.

## Decision

A **two-layer manifest**, mirroring the port:

- **Engine-neutral core (stable contract across adapters):** `run_id`; identity
  (`account` / `role-arn` / `region` — **never credentials**); resolved-config snapshot
  (for reproduction); per-dataset spec (`columns`, `where`, `order_by`, `limit`); masking
  **policy + masking applied per column**; row counts; local + staging paths; status;
  timestamps.
- **Namespaced `adapter: {…}` block (schema owned by each adapter):** for Athena —
  `query_execution_id`, `generated_sql`, `bytes_scanned`, `workgroup`, `staging_uri`.

**Masking proof = claim + evidence.** The core's *masking applied per column* is the
machine-checkable **claim** (and fail-closed, ADR-0003, guarantees applied ≡ policy or the
run aborted); the adapter block's `generated_sql` is the human-forensic **evidence**. A
future adapter supplies its own evidence; the claim record stays identical.

## Consequences

### Positive
- Manifest survives future adapters unchanged at the core; the Athena SQL is preserved as
  proof without polluting the contract.

### Negative / costs
- Masking is represented three ways (policy / applied / SQL). Acceptable: they serve three
  readers (config author / auditor-machine / auditor-human) and fail-closed keeps them
  consistent.

### Follow-ups (YAGNI for MVP)
- Parquet content hashes / SQL checksums for stronger reproducibility verification —
  deferred, not built now.

## Alternatives considered
- **Flat, Athena-shaped manifest**: rejected — leaks the engine; breaks on a second adapter.

## Related
- ADR-0001 (same two-layer idea), ADR-0003 (masking claim), ADR-0004 (bytes scanned).
