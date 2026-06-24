# CONTEXT.md — domain glossary for `ldt`

The canonical vocabulary for **Local Data Twin (`ldt`)**. Each term is defined once here and
links to the ADR that governs it. Other docs (`CLAUDE.md`, PRDs, issues) **reference** these
terms rather than redefining them — if a definition needs to change, change it here.

Single-context project: this file + [`docs/adr/`](docs/adr/) at the repo root.

## Core

- **Local Data Twin (`ldt`)** — a local CLI that builds a reproducible, governed local test
  environment from corporate tables registered in **AWS Glue Catalog** and stored in **S3**,
  regardless of whether the physical format is **Delta Lake** or **Apache Iceberg**. The local
  environment is *"production-shaped," not "production-equivalent."*
- **Governed Extract Local Twin** — the architectural pattern `ldt` implements: resolve identity
  → read the corporate catalog → extract through a governed port → stage in S3 → sync masked
  Parquet locally → expose via DuckDB → record in a manifest. The whole flow exists to get
  *safe, reproducible, governed* data onto a laptop.
- **Run / `run_id`** — one execution of `pull`. Each run gets a fresh `run_id` that namespaces
  its [staging prefix](#governance--safety) and its manifest/report, so runs never collide.
- **AWS identity** — the resolved caller (account / role-arn / region) from the user's ambient
  credentials (profile / SSO / SDK default chain) plus an optional assumed role. Recorded in
  the manifest **without credentials**. See [ADR-0007](docs/adr/0007-least-privilege-iam-documented-not-required.md).

## Extraction

- **Extraction port** — the coarse, engine-agnostic contract `extract(spec) -> LocalDataset`.
  The seam that makes `ldt` format-agnostic: orchestration depends only on the port, never on
  the engine. Format-agnosticism is the *driver* for the abstraction; governance and
  reproducibility are obligations the port imposes on every adapter.
  See [ADR-0001](docs/adr/0001-extraction-behind-coarse-engine-agnostic-port.md).
- **Adapter** — a concrete implementation behind the extraction port. All engine specifics
  (SQL, staging, polling, sync) live below the port line, inside an adapter.
- **Athena `UNLOAD` adapter** — the **sole MVP adapter**. Builds an
  `UNLOAD(SELECT … masking …) TO <staging prefix> WITH (format='PARQUET', compression='SNAPPY')`,
  runs it asynchronously, and polls to completion. Local Delta/Iceberg readers and any direct
  productive-S3 read are out of scope. See [ADR-0002](docs/adr/0002-athena-unload-sole-mvp-adapter.md).
- **Spec** — the per-dataset extraction request handed to the port: columns, `where`,
  `order_by`, `limit`, and the per-column masking map.
- **`LocalDataset`** — the port's return value: local Parquet paths plus extract metadata
  (masking applied per column, row counts) — the engine-neutral evidence the manifest records.

## Governance & safety

- **Staging prefix / staging zone** — the only S3 location `ldt` reads:
  `s3://<staging-bucket>/users/<aws_identity>/<project>/<run_id>/<dataset>/`. Output target of
  the `UNLOAD`; source of the local sync. The `<run_id>` keeps the target empty (Athena
  requires it). See [ADR-0001](docs/adr/0001-extraction-behind-coarse-engine-agnostic-port.md),
  [ADR-0002](docs/adr/0002-athena-unload-sole-mvp-adapter.md).
- **Enforcement by construction** — governance's primary teeth: no code path dereferences a
  source location; only the staging prefix is read. Backed by the **staging-only read test**.
- **Staging-only read test** — the test that asserts every S3 GET targets the staging prefix.
  The mechanical proof of "no productive-S3 reads."
  See [ADR-0001](docs/adr/0001-extraction-behind-coarse-engine-agnostic-port.md).
- **Least-privilege IAM** — the documented minimal permission contract (Glue read; Athena
  start/get/stop/get-workgroup; S3 read/write **only** on the user's staging subtree;
  deny-by-absence elsewhere). IAM is *reinforcement*, not the guarantee — under ambient creds it
  is defense-in-depth; an `assume_role_arn` upgrades it to a real boundary. **Lake Formation** is
  relied upon when present (the tool reads grants, never manages them) and absent gracefully.
  See [ADR-0007](docs/adr/0007-least-privilege-iam-documented-not-required.md).

## Masking

- **Masking** — transforming sensitive columns *before data lands locally*. The engine-neutral
  invariant is **masked-before-landing**; applying it inside the `UNLOAD` SQL is the Athena
  *mechanism*, not the invariant. See [ADR-0003](docs/adr/0003-fail-closed-masking-closed-type-set.md).
- **Fail-closed masking** — an unsatisfiable masking rule aborts the **whole dataset** (never
  partial, never unmasked), caught early in `validate`/`plan` and re-asserted in the adapter.
- **Masking type set** — a **closed, table-driven enum**: `hash`, `null`, `constant`,
  `date_bucket_month`. Modeled in Pydantic so an unknown type is a config error. **No plugins,
  no user-supplied SQL, ever.** `hash` → `sha256(CAST(<col> AS varchar)) AS <col>`.
- **Masking proof** — the audit guarantee that masking happened: a machine-checkable *claim* in
  the manifest core (applied-per-column) + *forensic evidence* (the generated SQL) in the
  adapter block. See [ADR-0005](docs/adr/0005-two-layer-manifest.md).

## Cost guardrail

Layered, per [ADR-0004](docs/adr/0004-cost-guardrail-where-limit-and-bytes-scanned.md):

- **Friction floor** — a mandatory `where` **or** `limit` on every dataset. Validation fails
  without one. It is **not** a cost ceiling — it only forces intent.
- **Bytes-scanned ceiling** — the real cost cap: an Athena `BytesScannedCutoffPerQuery` (and/or
  a per-run config cap) that aborts a query past a threshold.
- **Partition-prune warning** — a `plan`-time warning when a dataset's `where` touches no Glue
  partition key (so it won't prune and may scan the whole table). Uses Glue partition metadata
  only — no dry-run query, so `plan` stays free.

## Outputs

- **Two-layer manifest** — the per-run audit record (`manifests/<run_id>.json`):
  an **engine-neutral core** (run_id, identity sans credentials, resolved-config snapshot,
  per-dataset spec, masking policy + applied-per-column, row counts, paths, status, timestamps)
  + a namespaced **adapter block** (Athena: query execution id, generated SQL, bytes scanned,
  workgroup, staging uri). The core survives an adapter swap.
  See [ADR-0005](docs/adr/0005-two-layer-manifest.md).
- **Report** — a human-readable Markdown summary per run (`reports/<run_id>.md`).
- **DuckDB view layer** — one **view** per dataset over `read_parquet(...)`. The `.duckdb` holds
  view definitions only and is **disposable/regenerable** from manifest + Parquet. **Parquet is
  the single source of truth**; DuckDB is a downstream access layer outside the port.
  See [ADR-0006](docs/adr/0006-duckdb-views-over-parquet.md).
