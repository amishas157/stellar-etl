# H001: Transaction exports mirror `disk_read_bytes` into `read_bytes`

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For Soroban transactions, `soroban_resources_read_bytes` and `soroban_resources_disk_read_bytes` should preserve the two distinct counters from `SorobanTransactionData.Resources`. If a transaction reads bytes from memory without the same number of disk-read bytes, the JSON and Parquet exports should emit those different values exactly.

## Mechanism

`TransformTransaction()` never tracks `Resources.ReadBytes` separately. It defines only `outputSorobanResourcesDiskReadBytes`, then populates both exported fields from that one variable when building `TransactionOutput`, so the in-memory row is already corrupted before Parquet conversion runs. Downstream consumers therefore see `read_bytes == disk_read_bytes` for every Soroban transaction, even when the source XDR carried a different in-memory read count.

## Trigger

Run `export_transactions` on a Soroban transaction whose `SorobanTransactionData.Resources.ReadBytes` differs from `Resources.DiskReadBytes`. The exported row should contain the real read-byte count, but this path should instead repeat the disk-read-byte value in both columns.

## Target Code

- `internal/transform/transaction.go:137-140` - local Soroban resource counters omit a dedicated `ReadBytes` variable
- `internal/transform/transaction.go:162-165` - transform reads only `DiskReadBytes` and `WriteBytes` from the XDR resources
- `internal/transform/transaction.go:257-260` - `TransactionOutput` assigns both `SorobanResourcesReadBytes` and `SorobanResourcesDiskReadBytes` from `outputSorobanResourcesDiskReadBytes`
- `internal/transform/schema.go:71-74` - JSON schema exposes separate `soroban_resources_read_bytes` and `soroban_resources_disk_read_bytes` fields
- `internal/transform/parquet_converter.go:89-94` - Parquet faithfully copies the already-corrupted JSON struct, so both export surfaces inherit the same wrong value

## Evidence

The transform code declares `outputSorobanResourcesDiskReadBytes` but no sibling `outputSorobanResourcesReadBytes`, and the final struct literal uses the disk-read variable for both exported fields. That is a direct copy-paste mismatch between the XDR resource model and the emitted transaction schema.

## Anti-Evidence

This only becomes visible on Soroban transactions where `ReadBytes` and `DiskReadBytes` are not equal; classic transactions and Soroban transactions with matching counters will appear correct.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

The hypothesis assumes that `SorobanResources` in the XDR schema contains two separate fields: `ReadBytes` (in-memory reads) and `DiskReadBytes` (disk reads). I traced the actual XDR struct definition in the current SDK (`go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775`) and found that `SorobanResources` has only ONE read-bytes field: `DiskReadBytes`. There is no separate `ReadBytes` field. The older `stellar/go` SDK had a field named `ReadBytes` which was later renamed to `DiskReadBytes` — they are the same field, not two distinct counters.

### Code Paths Examined

- `go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:33922-33931` — `SorobanResources` struct has fields: `Footprint`, `Instructions`, `DiskReadBytes`, `WriteBytes`. No `ReadBytes` field exists.
- `go@v0.0.0-20240111173100-ed7ae81c8546/xdr/xdr_generated.go` — Older SDK version had `ReadBytes` (not `DiskReadBytes`), confirming it was a rename, not an addition.
- `internal/transform/transaction.go:139` — Declares `outputSorobanResourcesDiskReadBytes` as the only read-bytes variable.
- `internal/transform/transaction.go:164` — Reads `sorobanData.Resources.DiskReadBytes` (the only available field).
- `internal/transform/transaction.go:259-260` — Assigns `outputSorobanResourcesDiskReadBytes` to both `SorobanResourcesReadBytes` and `SorobanResourcesDiskReadBytes` output fields.
- `internal/transform/schema.go:72-73` — Schema defines two output fields, but both correctly contain the same value since the XDR only has one source field.

### Why It Failed

The hypothesis is based on a false premise. It assumes the XDR `SorobanResources` struct contains two distinct counters (`ReadBytes` for in-memory reads and `DiskReadBytes` for disk reads). In reality, the current XDR schema has only ONE field: `DiskReadBytes`. The field was renamed from `ReadBytes` to `DiskReadBytes` across SDK versions — they are the same semantic concept, not separate counters. The code correctly assigns the single XDR value to both output schema fields, which exist for backward compatibility with the old field name.

### Lesson Learned

Before hypothesizing about missing XDR field extraction, verify the actual XDR struct definition in the SDK version used by the project. Field renames across SDK versions can create the appearance of a "missing" field when the old name was simply replaced.
