# ADR-0001: Extraction behind a coarse, engine-agnostic port

- Status: Accepted
- Date: 2026-06-24
- Deciders: Jaime Casado

## Context

Extraction is the heart of `ldt`. No extraction contract exists yet (greenfield), so we
get to design the contract *before* the implementation.

Three forces push on this decision; their ranking is what rules the alternatives out:

1. **Format-agnosticism (primary driver).** The tool must work whether a corporate table
   is physically Delta or Iceberg. Because no contract exists, we prefer an
   **interface-first** approach: define the port now, treat any concrete engine as a
   swappable adapter behind it.
2. **Governance / least-exfiltration.** The tool must respect the corporate catalog and
   never become an exfiltration vector.
3. **Reproducibility / audit.**

Format-agnosticism is the load-bearing driver: it is *why* extraction is abstracted at
all. Governance and reproducibility are expressed as obligations the port imposes on every
adapter (see Decision), not as the reason for the abstraction.

## Decision

Define a single **coarse, engine-agnostic extraction port**:

```
extract(spec) -> LocalDataset      # local Parquet paths + extract metadata
```

Everything engine-specific — `UNLOAD` SQL generation, S3 staging, query polling, download
— lives **below the line** as adapter internals. The port is drawn at a high altitude on
purpose: a fine-grained port that exposed staging paths or query ids would leak the engine
and give only *fake* agnosticism.

Governance is encoded as **contract obligations on every adapter**, with teeth:

- **No reads of productive S3 — enforced by construction.** The codebase contains no S3
  object reader pointed at anything except the staging prefix it just wrote. There is no
  code path that dereferences a source table's location, and Glue `StorageDescriptor.Location`
  is never opened for bytes. Prevention is "the code that would exfiltrate does not exist."
- **Masked before landing.** Data must be masked before it lands locally. *How* a given
  adapter achieves this (Athena does it in-SQL) is an adapter mechanism; the engine-neutral
  invariant is "no unmasked sensitive bytes on local disk."

This negative constraint is **backed by a staging-only read test** that asserts every S3
GET targets the staging prefix.

IAM is *optional reinforcement* (→ ADR-0007), not the primary guarantee — the tool runs
with the user's ambient credentials, which may already permit prod-S3 reads. The manifest
is the audit trail (→ ADR-0005).

## Consequences

### Positive
- Genuine engine-agnosticism; a future governed adapter only has to land masked Parquet.
- The security claim ("low exfiltration risk") is real and testable, not prose.

### Negative / costs
- A port with a single adapter risks over-abstraction; mitigated by keeping it *coarse*
  (cheap, one method) rather than building an extension framework.
- The staging-only read test is load-bearing and must be maintained.

### Follow-ups
- "Masking in SQL" is therefore an Athena mechanism, not a universal invariant (→ ADR-0003).

## Alternatives considered
- **Local Delta/Iceberg readers over direct S3** (`deltalake`/`pyiceberg`): rejected —
  bypasses Glue/Lake Formation governance and defeats the security model (→ ADR-0002).
- **Fine-grained port** exposing staging/SQL/query seams: rejected — leaks Athena, yields
  fake agnosticism.
- **IAM as primary enforcement**: rejected — ambient user creds may already allow prod-S3.

## Related
- ADR-0002 (the one MVP adapter), ADR-0003 (masking), ADR-0005 (manifest), ADR-0007 (IAM).
