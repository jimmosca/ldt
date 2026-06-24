# ADR-0007: Least-privilege IAM — documented & role-recommended, not required

- Status: Accepted
- Date: 2026-06-24
- Deciders: Jaime Casado

## Context

The tool runs with the user's **ambient AWS credentials** (profile / SSO / default chain),
with optional `assume_role_arn`. A corporate analyst's identity almost certainly carries
**far more** than the tool needs — quite possibly prod-S3 read. So under ambient creds,
least-privilege is *aspirational*: IAM is **not** the enforcement boundary. ADR-0001
already makes enforcement-by-construction the primary teeth; this ADR must be honest about
IAM's lesser weight.

## Decision

- **Document the minimal policy** (the permission contract the tool needs):
  - **Glue (read-only):** `GetDatabase`, `GetTable`, `GetTables`, `GetPartitions`.
  - **Athena:** `StartQueryExecution`, `GetQueryExecution`, `StopQueryExecution`,
    `GetWorkGroup`.
  - **S3 — staging only:** `PutObject`, `GetObject`, `ListBucket`, `DeleteObject` scoped to
    `s3://<staging>/users/<identity>/*` (the `<identity>` prefix lets the policy pin each
    user to their own subtree).
  - **Deny by absence:** no read on productive S3, no Glue mutation, **no writes to source
    tables.**
- **`assume_role_arn` is the supported & recommended** way to actually achieve
  least-privilege — but **not required** in the MVP (running as the user is allowed).
- **State honestly:** under ambient creds, IAM is **defense-in-depth**; the primary teeth
  is enforcement-by-construction (ADR-0001). A scoped role upgrades IAM to a real boundary.
- **Lake Formation:** designed **LF-aware but not LF-dependent.** When the lake is
  LF-governed, LF grants — not the IAM policy — mediate column/row access; the tool may need
  `lakeformation:GetDataAccess` and **relies on** the user's LF/Glue grants being correct.
  The tool **never grants or manages** Lake Formation (consistent with the "no full Lake
  Formation management" scope boundary). It degrades to plain Glue+S3 IAM when LF is absent.

## Consequences

### Positive
- Low friction for MVP users; honest about what IAM does and doesn't guarantee.
- A clear, minimal policy and scoped-role path for security-conscious deployments.

### Negative / costs
- Running as the user means IAM provides no hard exfiltration guarantee on its own — by
  design, that job belongs to ADR-0001.

## Alternatives considered
- **Require a scoped role**: rejected — too much friction for the MVP.
- **IAM as primary enforcement**: rejected — ambient user creds may already permit prod-S3.

## Related
- ADR-0001 (primary enforcement), ADR-0002 (the adapter these permissions serve).
