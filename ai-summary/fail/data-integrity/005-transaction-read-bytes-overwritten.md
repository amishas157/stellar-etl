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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of ai-summary/fail/data-integrity/003-transaction-read-bytes-overwritten.md
**Failed At**: reviewer

### Trace Summary

Traced `TransformTransaction()` in `internal/transform/transaction.go` lines 131–270. Confirmed that only `outputSorobanResourcesDiskReadBytes` is declared (line 139), populated from `sorobanData.Resources.DiskReadBytes` (line 164), and assigned to both `SorobanResourcesReadBytes` and `SorobanResourcesDiskReadBytes` (lines 259-260). The XDR `SorobanResources` struct (verified via test usage at `test_variables_test.go:42`) defines `Footprint`, `Instructions`, `DiskReadBytes`, and `WriteBytes` — there is no separate `ReadBytes` field in the Stellar protocol.

### Code Paths Examined

- `internal/transform/transaction.go:139` — only `outputSorobanResourcesDiskReadBytes uint32` declared; no separate read-bytes variable
- `internal/transform/transaction.go:164` — populated from `sorobanData.Resources.DiskReadBytes`, the only available XDR source
- `internal/transform/transaction.go:259-260` — both `SorobanResourcesReadBytes` and `SorobanResourcesDiskReadBytes` assigned from the single variable
- `internal/transform/test_variables_test.go:42-47` — XDR `SorobanResources` struct shows no `ReadBytes` field

### Why It Failed

This is a duplicate of `ai-summary/fail/data-integrity/003-transaction-read-bytes-overwritten.md`, which was itself a duplicate of three prior investigations across other subsystems. The XDR `SorobanResources` struct has never defined a separate `ReadBytes` field — only `DiskReadBytes` exists. The `SorobanResourcesReadBytes` schema column is a backward-compatible alias for the same underlying `DiskReadBytes` value. Populating both columns from `DiskReadBytes` is the only correct behavior since there is only one XDR source field.

### Lesson Learned

This hypothesis has now been submitted at least five times. The Stellar XDR `SorobanResources` type only exposes `DiskReadBytes`, not a generic `ReadBytes`. When two schema columns share a single XDR source, verify the upstream type definition before concluding the duplication is a bug.
