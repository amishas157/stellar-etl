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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (different entity from bucket-list/live-state pattern in 003/004)
**Failed At**: reviewer

### Trace Summary

Traced `TransformTransaction()` in `internal/transform/transaction.go` lines 131–270. Confirmed that only `outputSorobanResourcesDiskReadBytes` is declared (line 139), populated from `sorobanData.Resources.DiskReadBytes` (line 164), and assigned to both `SorobanResourcesReadBytes` and `SorobanResourcesDiskReadBytes` (lines 259-260). Then inspected the upstream XDR `SorobanResources` struct definition in `go-stellar-sdk/xdr/xdr_generated.go:33912-33932`. The XDR struct defines exactly four fields: `Footprint LedgerFootprint`, `Instructions Uint32`, `DiskReadBytes Uint32`, and `WriteBytes Uint32`. There is **no** `ReadBytes` field anywhere in the XDR definition. The XDR comment explicitly qualifies `diskReadBytes` as "The maximum number of bytes this transaction can read from disk backed entries" — there is no generic/non-disk read-bytes concept in the protocol.

### Code Paths Examined

- `go-stellar-sdk/xdr/xdr_generated.go:33912-33932` — XDR `SorobanResources` struct defines only `Footprint`, `Instructions`, `DiskReadBytes`, and `WriteBytes`; no `ReadBytes` field exists in the Stellar protocol
- `internal/transform/transaction.go:139` — only `outputSorobanResourcesDiskReadBytes uint32` declared; no separate read-bytes variable
- `internal/transform/transaction.go:164` — populated from `sorobanData.Resources.DiskReadBytes`, the only available XDR source
- `internal/transform/transaction.go:259-260` — both `SorobanResourcesReadBytes` and `SorobanResourcesDiskReadBytes` assigned from the single variable
- `internal/transform/schema.go:72-73` — schema exposes both columns with distinct JSON tags
- `internal/transform/parquet_converter.go:91-92` — Parquet converter widens both to `int64` from the same source values

### Why It Failed

The hypothesis assumes the XDR protocol exposes a generic `ReadBytes` metric distinct from `DiskReadBytes`, and that the ETL incorrectly uses the disk-specific value for both. In reality, the Stellar XDR `SorobanResources` struct has never defined a separate `ReadBytes` field — only `DiskReadBytes` exists. The `SorobanResourcesReadBytes` schema column is a backward-compatible alias (or a more generic name) for the same underlying `DiskReadBytes` concept. Since there is only one XDR source field, populating both ETL columns from `DiskReadBytes` is the only correct behavior. The code is not producing wrong data — it is accurately reflecting the single available protocol value in two schema columns. This follows the same root cause pattern seen in the bucket-list/live-Soroban-state column duplication (fail 003, 004) but applied to transaction resources.

### Lesson Learned

When the ETL schema exposes two similarly-named columns (e.g., `read_bytes` vs `disk_read_bytes`), always verify whether the upstream XDR defines two distinct source fields before concluding that duplication is a bug. The Stellar protocol frequently uses qualified names (`disk_*`) for its only variant of a concept, and ETL schemas may retain a generic alias alongside the qualified name for downstream compatibility.
