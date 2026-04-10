# H005: `soroban_resources_read_bytes` is filled with disk-read bytes

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`soroban_resources_disk_read_bytes` should report `SorobanResources.DiskReadBytes`, and `soroban_resources_read_bytes` should remain zero/unset unless a distinct XDR source exists for it. For a Soroban transaction with `DiskReadBytes = 4096`, the ETL should not export `4096` into both columns.

## Mechanism

`TransformTransaction()` declares `outputSorobanResourcesReadBytes` but never assigns it. When building `TransactionOutput`, it writes `outputSorobanResourcesDiskReadBytes` into both `SorobanResourcesReadBytes` and `SorobanResourcesDiskReadBytes`, so the JSON and Parquet exports duplicate the disk-read counter under two different column names.

## Trigger

Export any Soroban transaction whose `SorobanTransactionData.Resources.DiskReadBytes` is non-zero.

## Target Code

- `internal/transform/transaction.go:137-170` — reads Soroban resource counters
- `internal/transform/transaction.go:233-261` — assigns the disk-read value into both output fields
- `internal/transform/schema.go:70-75` — defines both JSON columns
- `internal/transform/schema_parquet.go:61-66` — defines both Parquet columns
- `internal/transform/parquet_converter.go:89-94` — carries the duplicated values into Parquet

## Evidence

The local variable `outputSorobanResourcesReadBytes` is declared but never populated or used in the final struct literal. `go doc` for `xdr.SorobanResources` in the current SDK shows `Instructions`, `DiskReadBytes`, and `WriteBytes`, but no separate `ReadBytes` field that could justify populating two distinct columns with the same number.

## Anti-Evidence

If the duplicate column is a backward-compatibility alias, identical values might be intentional for some consumers. The schema still labels the fields differently, so downstream users can reasonably interpret them as separate metrics.
