# CLAUDE.md

Operational guidance for Claude Code working in this repo. **This file points; it does not
duplicate.** Architecture lives in [`docs/adr/`](docs/adr/) (authoritative — where this file and
an ADR disagree, the ADR wins). Domain vocabulary lives in [`CONTEXT.md`](CONTEXT.md). Read those
for the *why* and the *what*; this file is the *how to operate*.

## What this is

**Local Data Twin (`ldt`)** — a local CLI that builds a reproducible, governed local test
environment from corporate tables in **AWS Glue Catalog** / **S3**, whether the physical format is
**Delta** or **Iceberg**. Pattern: **Governed Extract Local Twin** (see [`CONTEXT.md`](CONTEXT.md)).

```
AWS creds → ldt CLI → Glue Catalog → [ extraction port → Athena UNLOAD adapter ] → S3 staging → local sync → Parquet + DuckDB views + manifest
```

## Status

Greenfield. No code yet — the PRD (GitHub issue #1) and the seven ADRs define the MVP. As modules
land, update the **commands** and **module layout** sections below with reality.

## Working the loop: RPIV

Every change moves through **Research → Plan → Implement → Verify**. The first three point
into the sections below; **Verify is a hard rule.**

- **Research** — read the ticket, find the seam, load the governing ADR, the `CONTEXT.md`
  term, and the **Hard invariants** the change touches (see **Agent skills** → domain docs +
  issue tracker). When a specific aspect needs clarifying — an unfamiliar AWS behaviour, a
  library contract, the state of the art — **spin up a subagent to research it** before
  deciding.
- **Plan** — decide where the change lands relative to the extraction port line (above =
  orchestration/CLI; below = adapter internals) and whether it stays inside the closed
  masking set / manifest core. *The **Plan** stage ≠ `ldt plan` the command, which is a
  runtime cost-safety preview an operator runs.*
- **Implement** — build it per **Stack & commands**, keeping Athena specifics below the port
  and masking table-driven.
- **Verify** — **a change is not done until it is tested and checked** (lint + tests over the
  affected parts). The concrete runbook is **not yet defined** — greenfield, no toolchain —
  and is tracked in [#11](https://github.com/jimmosca/ldt/issues/11). Until it lands: never
  report a change verified against an empty or absent test suite — state what you ran, or that
  nothing exists to run yet.

## Hard invariants (do not break)

One-line rule each; the linked ADR owns the rationale — **read it before touching the behaviour.**

- **No productive-S3 reads** — only the staging prefix is read, enforced by construction + a staging-only read test. ([ADR-0001](docs/adr/0001-extraction-behind-coarse-engine-agnostic-port.md), [ADR-0007](docs/adr/0007-least-privilege-iam-documented-not-required.md))
- **No writes to source tables** — Glue read + Athena query + read/write on the tool's own staging prefix only; deny-by-absence. ([ADR-0007](docs/adr/0007-least-privilege-iam-documented-not-required.md))
- **Masked before it lands locally** — fail-closed (unsatisfiable rule aborts the whole dataset), closed table-driven type set, no user SQL. ([ADR-0003](docs/adr/0003-fail-closed-masking-closed-type-set.md))
- **Extraction stays behind the coarse port** — `extract(spec) -> LocalDataset`; Athena specifics never leak above the port line. ([ADR-0001](docs/adr/0001-extraction-behind-coarse-engine-agnostic-port.md), [ADR-0002](docs/adr/0002-athena-unload-sole-mvp-adapter.md))
- **Layered cost guardrail** — `where`/`limit` friction floor (not a ceiling) + bytes-scanned cutoff + `plan`-time partition-prune warning. ([ADR-0004](docs/adr/0004-cost-guardrail-where-limit-and-bytes-scanned.md))
- **Two-layer manifest** — engine-neutral core + namespaced adapter block; never contains credentials. ([ADR-0005](docs/adr/0005-two-layer-manifest.md))
- **DuckDB = disposable views over Parquet** — Parquet is the single source of truth. ([ADR-0006](docs/adr/0006-duckdb-views-over-parquet.md))

## CLI surface

Five commands. `pull` is the orchestrator; `plan` before `pull` is the core safety affordance —
keep it cheap and side-effect-free.

| Command | Job |
|---|---|
| `ldt init` | Scaffold `local-data-twin.yml` + `.data/` |
| `ldt validate -c <cfg>` | Check YAML, AWS creds, region, Glue db/tables/columns, staging bucket, Athena workgroup |
| `ldt plan -c <cfg>` | Print what would run — **no chargeable AWS calls** |
| `ldt pull -c <cfg>` | identity → Glue → masking/columns → extraction port → S3 sync → DuckDB views → manifest → report |
| `ldt open` | Open DuckDB CLI / print connection instructions |

The `pull` ordering is a contract: masking/column choices are decided **before** anything touches
S3 or local disk, then baked into the adapter's SQL.

## Intended module layout

Package `local_data_twin/`. The port/adapter seam ([ADR-0001](docs/adr/0001-extraction-behind-coarse-engine-agnostic-port.md)/[0002](docs/adr/0002-athena-unload-sole-mvp-adapter.md))
is fixed; names finalise during implementation.

| Module | Responsibility |
|---|---|
| `cli.py` | Typer commands |
| `config.py` | YAML load + Pydantic validation (closed masking enum here) |
| `aws_identity.py` | resolve profile/region/account/role (+ optional AssumeRole) |
| `glue_catalog.py` | table/column/partition metadata + column validation |
| `extract.py` | the coarse extraction **port** |
| `athena_export.py` | the Athena **adapter**: build `UNLOAD` SQL, run, poll |
| `s3_sync.py` | download Parquet from staging (staging-only reads), optional cleanup |
| `duckdb_register.py` | one view per dataset over `read_parquet(...)` |
| `masking.py` | table-driven masking rules → SQL expressions |
| `manifest.py` | write `manifests/<run_id>.json` (core + adapter block) |
| `report.py` | write `reports/<run_id>.md` |

## Config & output

Config keys: `project`, `aws` (profile/region/workgroup/assume_role_arn), `catalog` (glue +
database), `staging` (bucket/prefix_template/cleanup_after_download), `local`
(path/duckdb_path/lake_path), `datasets[]` (per-dataset: `columns`, `where`, `order_by`, `limit`,
column→masking map). The generated `UNLOAD` SQL shape is an adapter detail — see
[ADR-0002](docs/adr/0002-athena-unload-sole-mvp-adapter.md).

```
.data/
  local.duckdb
  lake/<dataset>/part-*.parquet
  manifests/<run_id>.json
  reports/<run_id>.md
```

## Stack & commands

Python · Typer · Pydantic · boto3 · duckdb · PyYAML/ruamel.yaml · Rich. Tooling: **uv**, **pytest**.

- Install: `uv sync`
- Run CLI: `uv run ldt <command>`
- Tests: `uv run pytest` / single: `uv run pytest tests/test_x.py::test_name`

Testing note: moto/localstack only if it pays off; for Athena/Glue, controlled integration tests
beat mocks. Test seams are fixed in the PRD (CLI↔fake-adapter integration; pure SQL-builder unit;
gated live integration).

## Scope boundaries (MVP)

Out of scope without an explicit ask: local Delta/Iceberg readers, direct prod-S3 reads, CDC,
incremental sync, write-back to S3, entity-graph sampling, UI portal, DataHub, multi-cloud,
editing source tables, full Lake Formation management, production-equivalence guarantees.

## Agent skills

### Issue tracker

GitHub issues in `jimmosca/ldt` via the `gh` CLI. External PRs are **not** a triage surface. See `docs/agents/issue-tracker.md`.

### Triage labels

Canonical defaults: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: [`CONTEXT.md`](CONTEXT.md) + [`docs/adr/`](docs/adr/) at the repo root. See `docs/agents/domain.md`.
