# H002: `StreamChanges()` Merges `batchSize+1` Ledgers into One Oversized Batch

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: Medium
**Impact**: Operational correctness: batch files cover more ledgers than the caller-requested batch size
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_ledger_entry_changes --batch-size N` should never emit a `ChangeBatch` covering more than `N` ledgers. For example, `--batch-size 1` over a two-ledger range should produce two single-ledger batches/files, and `--batch-size 64` over `1..65` should split into `1..64` plus `65..65`.

## Mechanism

`StreamChanges()` initializes `batchEnd` as `batchStart + batchSize`, which is an exclusive-style boundary, but then passes that value into `ExtractBatch()` as an inclusive end unless `batchEnd < end`. When the remaining range length is exactly `batchSize+1`, that decrement never happens, so `ExtractBatch()` receives an inclusive `[batchStart, batchEnd]` span containing `N+1` ledgers. The command then exports one larger file for that merged range, silently violating the requested batch partitioning.

## Trigger

1. Run `stellar-etl export_ledger_entry_changes -b 1 -s <L> -e <L+1> ...`.
2. Expected result: two batch files, one per ledger.
3. Actual result: a single batch covering both ledgers.

The same defect appears with the default batch size: `-b 64 -s 1 -e 65` should split after ledger `64`, but the current code path emits one `1..65` batch instead.

## Target Code

- `internal/input/changes.go:162-175` — computes `batchEnd` with mixed exclusive/inclusive semantics, then passes it to `ExtractBatch()`
- `internal/input/changes_test.go:159-179` — codifies the oversized `1..65` single-batch behavior for `batchSize == 64`
- `cmd/export_ledger_entry_changes.go:23-24` — user-facing contract says batching is controlled by the `batch-size` flag
- `cmd/export_ledger_entry_changes.go:274-285` — exports each computed batch directly as a distinct artifact
- `cmd/export_ledger_entry_changes.go:306-310` — filenames are derived from the oversized batch boundaries, so the wrong partitioning becomes persistent output
- `internal/utils/main.go:265-272` — flag text says `batch-size` is the number of ledgers to export changes from in each batch

## Evidence

The implementation's own tests already show the off-by-one shape: for `batchStart: 1`, `batchEnd: 65`, `batchSize: 64`, `TestStreamChangesBatchNumbers` expects one batch `1..65`, not a `64+1` split. That matches the code path where `batchEnd == end` skips the `batchEnd = batchEnd - 1` adjustment even though `ExtractBatch()` interprets `batchEnd` inclusively.

## Anti-Evidence

The rows inside the merged batch are still sourced from real ledgers, so this is not per-row field corruption. The bug is in caller-visible batching semantics and file partitioning: consumers expecting `N`-ledger batches receive larger artifacts than requested.
