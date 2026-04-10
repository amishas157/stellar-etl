# H004: Transaction `soroban_resources_read_bytes` silently duplicates disk read bytes

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`soroban_resources_read_bytes` should represent a distinct read-bytes metric if one exists in the XDR source, or else stay zero/absent when the on-chain metadata only exposes disk-backed reads. `soroban_resources_disk_read_bytes` should continue to hold the actual `diskReadBytes` value from `SorobanResources`.

## Mechanism

`TransformTransaction()` declares only `outputSorobanResourcesDiskReadBytes`, populates it from `sorobanData.Resources.DiskReadBytes`, and then writes that same variable into both `SorobanResourcesReadBytes` and `SorobanResourcesDiskReadBytes`. The schema and Parquet output therefore advertise two differently named resource columns even though one is just a copy of the other.

## Trigger

Export any Soroban transaction with non-zero `diskReadBytes`. The resulting JSON and Parquet rows will contain identical non-zero values in both `soroban_resources_read_bytes` and `soroban_resources_disk_read_bytes`.

## Target Code

- `internal/transform/transaction.go:TransformTransaction:137-165` — reads only `DiskReadBytes` from `SorobanResources`
- `internal/transform/transaction.go:TransformTransaction:257-260` — writes the disk-read value into both output columns
- `internal/transform/schema.go:TransactionOutput:69-75` — schema exposes separate `soroban_resources_read_bytes` and `soroban_resources_disk_read_bytes` fields
- `internal/transform/schema_parquet.go:TransactionOutputParquet:61-66` — Parquet schema preserves the same two-column distinction

## Evidence

The current SDK `xdr.SorobanResources` surface has `Instructions`, `DiskReadBytes`, and `WriteBytes`, but no separate generic `ReadBytes` field. Inside `TransformTransaction()`, line 164 assigns only `sorobanData.Resources.DiskReadBytes`, and lines 259-260 copy that one value into both exported resource-byte columns.

## Anti-Evidence

The duplicate field may have been kept as a legacy alias for downstream compatibility. But as exported today it is indistinguishable from a real source-backed metric, so downstream analytics can easily interpret one on-chain value as two separate resource budgets.
