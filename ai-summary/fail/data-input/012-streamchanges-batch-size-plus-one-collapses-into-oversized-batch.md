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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — substantially overlaps with fail/data-input/003-streamchanges-boundaries-are-non-overlapping.md (same batch boundary math analysis)
**Failed At**: reviewer

### Trace Summary

Traced `StreamChanges()` (changes.go:162-178) through multiple input scenarios. When `(end - start + 1) % batchSize == 1`, `batchEnd` equals `end`, the `if batchEnd < end` guard on line 166 is false, so the decrement is skipped and `ExtractBatch` receives an inclusive range spanning `batchSize + 1` ledgers. However, the test suite's "single" case (`start=1, end=65, batchSize=64` → one batch `[1,65]`) explicitly codifies this behavior as intended. The `exportTransformedData` function (export_ledger_entry_changes.go:306-308) includes a comment acknowledging the inclusive end convention and adjusts filenames accordingly with `end+1`.

### Code Paths Examined

- `internal/input/changes.go:162-178` — `StreamChanges` loop: confirmed `batchEnd == end` skips decrement, producing `batchSize+1` inclusive range
- `internal/input/changes.go:82-158` — `extractBatch`: loop uses `seq <= batchEnd` (inclusive), confirming it processes all `batchSize+1` ledgers
- `internal/input/changes_test.go:159-168` — "single" test case: explicitly expects `[1,65]` as one batch for `batchSize=64`, proving this is designed behavior
- `internal/input/changes_test.go:169-179` — "one extra" test case: `end=66` correctly splits into `[1,64]` + `[65,66]`, showing the decrement works for non-edge cases
- `cmd/export_ledger_entry_changes.go:304-310` — `exportTransformedData`: comment on lines 306-308 explicitly acknowledges inclusive `BatchEnd` and compensates in filename generation

### Why It Failed

This describes **working-as-designed behavior**, not a bug. The `StreamChanges` test suite explicitly codifies the `batchSize+1` merge for the boundary case, and the export command's filename generation includes a comment acknowledging the inclusive convention. The code deliberately avoids creating a 1-ledger remainder batch by absorbing it into the preceding batch. All ledger data is exported exactly once with no duplication or omission — the only difference is the partition boundary, which has no data correctness impact. Additionally, this is substantially the same batch-boundary-math investigation as fail/003, which already concluded the arithmetic is "odd but internally consistent."

### Lesson Learned

When existing tests explicitly codify a batching behavior and the code includes comments acknowledging the convention, the behavior is intentional design — not a latent bug. Batch-size overflow of exactly 1 is a common deliberate choice to avoid degenerate single-element remainder batches.
