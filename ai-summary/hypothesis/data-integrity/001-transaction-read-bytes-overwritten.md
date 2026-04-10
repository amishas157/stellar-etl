# H001: Transaction export overwrites `soroban_resources_read_bytes` with disk-read bytes

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For Soroban transactions, `soroban_resources_read_bytes` should export `SorobanTransactionData.Resources.ReadBytes`, while `soroban_resources_disk_read_bytes` should export `SorobanTransactionData.Resources.DiskReadBytes`. When a transaction reads some footprint entries from memory and only a subset from disk, the two output columns should differ accordingly in both JSON and Parquet exports.

## Mechanism

`TransformTransaction()` never stores `Resources.ReadBytes` in its own variable. It only captures `DiskReadBytes`, then assigns that same value into both `SorobanResourcesReadBytes` and `SorobanResourcesDiskReadBytes` on the output struct. Any Soroban transaction where `ReadBytes != DiskReadBytes` therefore exports a plausible-looking but wrong resource profile, which breaks downstream resource-usage analytics and fee-model reconciliation.

## Trigger

Export any Soroban transaction whose `SorobanTransactionData.Resources.ReadBytes` is larger than `Resources.DiskReadBytes` because some reads are served from the in-memory footprint. The exported row will show `soroban_resources_read_bytes == soroban_resources_disk_read_bytes` instead of the true read-byte count.

## Target Code

- `internal/transform/transaction.go:135-170` — captures Soroban resource counters, but only declares/populates `outputSorobanResourcesDiskReadBytes`
- `internal/transform/transaction.go:233-261` — writes `outputSorobanResourcesDiskReadBytes` into both exported fields
- `internal/transform/schema.go:70-75` — schema defines separate `soroban_resources_read_bytes` and `soroban_resources_disk_read_bytes` columns
- `internal/transform/parquet_converter.go:89-94` — Parquet conversion faithfully propagates the already-corrupted JSON field

## Evidence

The local variable list in `TransformTransaction()` includes `outputSorobanResourcesDiskReadBytes` and `outputSorobanResourcesWriteBytes`, but no `outputSorobanResourcesReadBytes`. Later, the struct literal sets `SorobanResourcesReadBytes: outputSorobanResourcesDiskReadBytes` and `SorobanResourcesDiskReadBytes: outputSorobanResourcesDiskReadBytes`, making the duplication unconditional.

## Anti-Evidence

If a transaction happens to have `ReadBytes == DiskReadBytes`, the corruption is not visible. There is also no later normalization step that could repair the value, so the bug only hides on naturally equal inputs.
