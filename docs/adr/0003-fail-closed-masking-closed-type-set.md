# ADR-0003: Fail-closed masking with a closed, table-driven type set

- Status: Accepted
- Date: 2026-06-24
- Deciders: Jaime Casado

## Context

The engine-neutral invariant from ADR-0001 is "data is masked before it lands locally."
That invariant is only meaningful if we pin what happens when masking **cannot** be fully
applied: an unknown masking type (`hsah`), a column not present in the selected projection,
or a type a rule cannot express. For a tool whose pitch is "sensitive values never land
locally unmasked," the failure mode is the whole ballgame.

## Decision

**Fail-closed, and fail early.** Any masking rule that cannot be fully realized **aborts
the entire dataset extract** before any bytes move — never partial, never silently skipped.

- Validated **statically** as early as possible: `validate` checks the column exists in
  Glue; `plan` checks the masking type is known and the column is in the projection (both
  cheap, no AWS cost). The adapter **re-asserts** immediately before running as a backstop.
- The masking **type set is a closed enum** (`hash`, `null`, `constant`, `date_bucket_month`).
  Implemented in Pydantic as a `Literal`/`Enum`, so an unknown type is a *config validation
  error* — fail-closed for free. `hash` → `sha256(CAST(<col> AS varchar)) AS <col>`.
- **Table-driven** internally: a dispatch table `{MaskType -> SQL-expression builder}`.
  Adding a type later is one enum member + one small function + one table row — a tiny,
  reviewed PR. **No plugins, no user-supplied SQL, ever.**
- The manifest records **masking applied per column** as the audit claim (→ ADR-0005).

## Consequences

### Positive
- Sensitive data can never land partially or unmasked.
- A small, reviewed masking vocabulary keeps the audit story trustworthy.
- Adding a masking type is cheap despite the set being closed.

### Negative / costs
- Users cannot add masking types without a code change (intended — it's a security surface).

## Alternatives considered
- **Fail-open** (extract what you can, skip unsatisfiable rules): rejected — catastrophic;
  sensitive values land unmasked and unnoticed.
- **Pluggable / user-supplied SQL masking**: rejected — reopens the fail-closed hole, is a
  SQL-injection surface, and breaks the manifest's ability to certify masking.

## Related
- ADR-0001 (the invariant this enforces), ADR-0005 (masking recorded as proof).
