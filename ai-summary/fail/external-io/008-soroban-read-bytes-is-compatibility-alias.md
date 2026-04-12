# H008: `soroban_resources_read_bytes` should diverge from `disk_read_bytes`

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the schema's `soroban_resources_read_bytes` and `soroban_resources_disk_read_bytes` columns represent distinct on-chain resource counters, the exporter should source them from different XDR fields and preserve different values when the transaction data differs.

## Mechanism

I investigated whether `TransformTransaction()` was silently duplicating `disk_read_bytes` into both output columns, which would make one of the Soroban resource metrics wrong in JSON and Parquet exports. That would be a concrete copy-paste bug if the current XDR still exposed separate `readBytes` and `diskReadBytes` values.

## Trigger

Export a Soroban transaction whose resource metadata would need distinct read-byte counters and compare the JSON/Parquet fields.

## Target Code

- `internal/transform/transaction.go:139-165` — only populates `outputSorobanResourcesDiskReadBytes`
- `internal/transform/transaction.go:258-260` — writes the same value into both exported columns
- `internal/transform/schema.go:71-74` — exposes both `soroban_resources_read_bytes` and `soroban_resources_disk_read_bytes`
- `internal/transform/schema_parquet.go:62-65` — exposes both columns in Parquet as well

## Evidence

The transform code really does assign `outputSorobanResourcesDiskReadBytes` to both exported fields, so the duplication looks suspicious in isolation.

## Anti-Evidence

After checking the current generated XDR definitions, `xdr.SorobanResources` only exposes `Instructions`, `DiskReadBytes`, and `WriteBytes`; there is no distinct `ReadBytes` source field left to export. That means the duplicated column is a compatibility alias, not a live corruption path fed by separate on-chain values.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The current XDR schema no longer contains a separate `readBytes` counter, so there is no missed source value for the exporter to preserve. The duplicated export field is misleading, but it does not currently overwrite a distinct on-chain metric.

### Lesson Learned

For schema-audit findings, always verify that the upstream XDR still carries a distinct source field. A suspicious duplicate output column is not enough by itself if protocol evolution already collapsed the underlying data model.
