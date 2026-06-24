# Handoff — `ldt`: cut issues from the MVP PRD

**Next session's job:** run `/to-issues` to split the PRD into independently-grabbable issues, in a clean context. This window got long (setup → ADR grilling → 7 ADRs → CLAUDE.md reconcile → PRD), so we're handing off *before* `/to-issues` to keep it inside the smart zone. Everything needed is on disk / on GitHub — a cold read loses nothing.

## Repo

- `/home/casadja/projects/ldt` — greenfield, **no code yet**. The agentic surface lives on branch `docs/agentic-surface` (not yet merged to `main`). GitHub remote `jimmosca/ldt`.
- `gh` is authenticated as `jimmosca` (HTTPS). The `ready-for-agent` label exists.

## Authoritative artifacts (read these; don't re-derive)

- **PRD → GitHub issue #1**: https://github.com/jimmosca/ldt/issues/1 — problem/solution, 33 user stories, implementation decisions mapped to the ADRs, the confirmed test seams, scope boundaries. **This is the input to `/to-issues`.**
- **ADRs → `docs/adr/` (0001–0007, see `docs/adr/README.md`)** — authoritative architecture. The hard ones to respect when slicing:
  - 0001 coarse engine-agnostic extraction port `extract(spec) -> LocalDataset`; governance enforced by construction + a staging-only read test.
  - 0002 Athena `UNLOAD` is the sole MVP adapter; no local Delta/Iceberg or direct prod-S3 readers.
  - 0003 fail-closed masking, closed table-driven type set.
  - 0004 layered cost guardrail (`where`/`limit` floor + bytes-scanned cutoff + prune warning).
  - 0005 two-layer manifest (engine-neutral core + adapter block).
  - 0006 DuckDB views over Parquet, disposable.
  - 0007 least-privilege IAM, documented & role-recommended; LF relied-upon, not managed.
- **`CONTEXT.md`** — canonical domain glossary (Governed Extract Local Twin, extraction port/adapter, LocalDataset, staging prefix, fail-closed masking, friction-floor-vs-bytes-scanned-ceiling, two-layer manifest, etc.), each term linking to its governing ADR. **Use these terms in issues; don't re-derive vocabulary from the PRD.**
- **`CLAUDE.md`** — compacted to a thin operational pointer; defers to `docs/adr/` (architecture) and `CONTEXT.md` (vocabulary) on any conflict.
- **`docs/agents/`** — issue-tracker.md (GitHub via `gh`, PRs *not* a triage surface), triage-labels.md (canonical defaults), domain.md (single-context).

> Surface state: the whole `docs/` tree + `CONTEXT.md` + the compacted `CLAUDE.md` are committed on branch `docs/agentic-surface` (was previously untracked). Merge or branch from there.

## Guidance for `/to-issues`

- The PRD already specifies the **test seams** — issues should align to them: a foundational **fake-extraction-adapter + fake-Glue/identity fixture** is likely an early, dependency-unblocking issue (Seam 1 depends on it). The **Athena `UNLOAD` SQL builder** is a self-contained pure-function issue (Seam 2).
- The **port/adapter seam (ADR-0001/0002) is fixed** — don't let an issue collapse Athena specifics into the port.
- Natural dependency spine: config+Pydantic models (incl. closed masking enum) → Glue catalog access → extraction port + fake adapter → Athena adapter (SQL builder, run/poll) → S3 sync (staging-only) → DuckDB views → manifest → report → CLI wiring (`init/validate/plan/pull/open`). `/to-issues` owns the actual cut; this is just orientation.
- Apply the `ready-for-agent` label to each issue (no further triage — these aren't raw incoming requests).

## Not done (optional, don't block on these)

- _(none outstanding — the `CONTEXT.md` glossary that was deferred is now seeded and committed.)_

## Suggested skills

1. **`/to-issues`** — primary task. Feed it issue #1 + the ADRs.
2. After issues are cut: **clear context and run `/implement` once per issue** in a fresh session, passing the PRD (#1) + the single issue.
3. **Skip `/setup-matt-pocock-skills`** — already done this session (tracker/labels/domain configured).
