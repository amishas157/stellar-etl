# H003: `soroban_resources_read_bytes` is populated from `disk_read_bytes`

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Duplicate resource metrics
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`soroban_resources_read_bytes` should either come from a distinct on-chain source or remain zero/empty when that source does not exist. It should not be silently cloned from `soroban_resources_disk_read_bytes` unless the XDR model explicitly defines them as the same metric.

## Mechanism

`TransformTransaction()` stores `sorobanData.Resources.DiskReadBytes` in both `SorobanResourcesReadBytes` and `SorobanResourcesDiskReadBytes`. That makes the exported row claim two identical resource measurements even though the current XDR type only exposes one disk-read byte counter.

## Trigger

Export any Soroban transaction with non-zero `sorobanData.Resources.DiskReadBytes`.

## Target Code

- `internal/transform/transaction.go:257-260` — both output columns are sourced from the same `outputSorobanResourcesDiskReadBytes` variable
- `internal/transform/schema.go:70-75` — the JSON schema models `read_bytes` and `disk_read_bytes` as separate fields

## Evidence

The transformer never reads a separate source value before populating `SorobanResourcesReadBytes`. The current XDR `SorobanResources` type exposes `Instructions`, `DiskReadBytes`, and `WriteBytes`, but not an independent read-bytes counter.

## Anti-Evidence

Because the current XDR model does not provide a second source metric, I could not prove a distinct correct on-chain value that the exporter should have emitted instead. This may be legacy schema debt rather than a concrete repo-local corruption bug with a verifiable replacement value.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The duplicate-column behavior is suspicious, but there is no current XDR field to establish what `soroban_resources_read_bytes` should be for the same transaction. Without a concrete replacement source, I could not frame it as a testable wrong-value hypothesis instead of a schema-design concern.

### Lesson Learned

For type and mapping audits, a duplicated output column is only a viable data-integrity finding when the source of truth for the missing value is known and reachable from the current exporter.
