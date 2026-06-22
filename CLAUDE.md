# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status

Greenfield. No code yet — the repository is empty. The MVP spec lives in the original `/init` handoff and is summarized below; treat it as the source of truth until code exists. As modules land, update this file with the real commands.

## What this is

**Local Data Twin (`ldt`)** — a local CLI that builds a reproducible, governed local test environment from corporate tables registered in **AWS Glue Catalog** and stored in **S3**, regardless of whether the physical format is **Delta Lake** or **Apache Iceberg**.

Product line: *"Generate a local, safe, reproducible data environment using your AWS credentials and respecting the corporate catalog."*

## The one architectural decision that drives everything

**All extraction goes through Athena `UNLOAD` → S3 staging → local sync.** The CLI must **never** read productive data directly from S3 (e.g. `s3://production-lake/...`). It reads only the exported result in a controlled staging zone:

```
s3://<staging-bucket>/users/<aws_identity>/<project>/<run_id>/<dataset>/
```

This is what makes the tool format-agnostic (Athena abstracts Delta vs Iceberg), governed (respects Glue + corporate permissions), and low-exfiltration-risk. Do **not** introduce local Delta/Iceberg readers — that path is explicitly out of scope for the MVP and would defeat the security model.

This pattern is called **Governed Extract Local Twin**. Full flow:

```
AWS user credentials → ldt CLI → Glue Catalog → Athena UNLOAD → S3 staging → local sync → Parquet + DuckDB + manifest
```

## Hard invariants (do not break these)

- No direct read of productive S3 paths from local. Only Athena `UNLOAD` output.
- No writes to source tables. The tool only needs read on Glue metadata + Athena query + read/write on its **own** result/staging buckets.
- Every dataset must have a `where` or a `limit` (cost guardrail — an unfiltered `UNLOAD` can scan the whole table).
- Masking is applied **inside the generated SQL** (the `UNLOAD` SELECT), never after download — so sensitive values never land locally unmasked.
- The `manifest.json` must capture enough to reproduce/audit a sample (identity, query params, columns, masking, paths, status).

## CLI surface

Five commands. `pull` is the orchestrator.

| Command | Job |
|---|---|
| `ldt init` | Scaffold `local-data-twin.yml` + `.data/` |
| `ldt validate -c <cfg>` | Check YAML, AWS creds, region, Glue db/tables/columns exist, staging bucket accessible, Athena workgroup usable |
| `ldt plan -c <cfg>` | Print what would run (datasets, columns, filters, limits, masking) — **no AWS calls that cost money** |
| `ldt pull -c <cfg>` | Resolve identity → query Glue → generate `UNLOAD` SQL → run Athena → wait → download Parquet → register DuckDB views → write manifest → write report |
| `ldt open` | Open DuckDB CLI / print connection instructions |

`plan` before `pull` is a core safety affordance — keep it cheap and side-effect-free.

## Intended module layout & responsibilities

Package `local_data_twin/`:

- `cli.py` — Typer commands (`init/validate/plan/pull/open`)
- `config.py` — load YAML, validate with **Pydantic**
- `aws_identity.py` — resolve profile/region/account/role (supports `AWS_PROFILE`, SSO, SDK default chain, optional `AssumeRole`)
- `glue_catalog.py` — `get_table`/`list_tables`, validate requested columns against schema
- `athena_export.py` — build `UNLOAD` SQL, run query, poll to completion
- `s3_sync.py` — download Parquet from staging to `.data/lake/<dataset>/`, optional cleanup
- `duckdb_register.py` — create `.data/local.duckdb`, one view per dataset over `read_parquet(...)`
- `masking.py` — translate declarative masking rules → SQL expressions
- `manifest.py` — write `manifests/<run_id>.json`
- `report.py` — write `reports/<run_id>.md`

The `pull` flow ordering above is a contract — masking/columns are decided in `glue_catalog` + `masking` and baked into SQL by `athena_export` before anything touches S3 or local disk.

## Config & generated SQL contract

Config keys: `project`, `aws` (profile/region/workgroup/assume_role_arn), `catalog` (glue + database), `staging` (bucket/prefix_template/cleanup_after_download), `local` (path/duckdb_path/lake_path), `datasets[]`.

Per-dataset sampling: `columns` (explicit), `where`, `order_by`, `limit`. Masking maps a column → one of: `hash`, `null`, `constant`, `date_bucket_month`.

`hash` → `sha256(CAST(<col> AS varchar)) AS <col>`. Generated query shape:

```sql
UNLOAD (
  SELECT order_id, sha256(CAST(customer_id AS varchar)) AS customer_id, order_date, region, amount, status
  FROM analytics_silver.orders
  WHERE order_date >= DATE '2026-01-01'
  ORDER BY order_date DESC
  LIMIT 100000
)
TO 's3://.../users/<identity>/<project>/<run_id>/orders/'
WITH (format = 'PARQUET', compression = 'SNAPPY');
```

Local output after `pull`:

```
.data/
  local.duckdb
  lake/<dataset>/part-*.parquet
  manifests/<run_id>.json
  reports/<run_id>.md
```

## Recommended stack

Python · Typer (CLI) · Pydantic (config) · boto3 (AWS) · duckdb · PyYAML/ruamel.yaml · Rich (output). Tooling: **uv** for env, **pytest** for tests.

Once set up, expected commands will be roughly:
- Install: `uv sync`
- Run CLI: `uv run ldt <command>` (or install as console_script entry point)
- Tests: `uv run pytest` / single test: `uv run pytest tests/test_x.py::test_name`

Testing note from the spec: moto/localstack only if it pays off; for Athena/Glue, controlled integration tests are likely more realistic than mocks.

## Scope boundaries (MVP)

Out of scope — do not build these without an explicit ask: local Delta/Iceberg readers, CDC, incremental sync, write-back to S3, entity-graph sampling, UI portal, DataHub, multi-cloud, editing source tables, full Lake Formation management, production-equivalence guarantees.

Framing: the local environment is **"production-shaped," not "production-equivalent."** DuckDB-over-Parquet does not reproduce Athena/Spark/Delta/Iceberg semantics exactly.
