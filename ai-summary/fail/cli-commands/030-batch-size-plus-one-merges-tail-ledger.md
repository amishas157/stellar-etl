# H002: `export_ledger_entry_changes` emits `batch_size+1` ledgers in one batch

**Date**: 2026-04-12
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: structural batch-partition corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_ledger_entry_changes` should never emit more than `--batch-size` ledgers in a single batch. For the default `--batch-size 64`, a 65-ledger range should produce two output-file sets per resource: one 64-ledger batch and one 1-ledger tail batch.

## Mechanism

`StreamChanges()` computes `batchEnd := min(batchStart+batchSize, end)` and only decrements `batchEnd` when that value is still strictly less than `end`. When the requested range length is exactly `batchSize+1`, the first `batchEnd` lands exactly on `end`, so no decrement happens and `ExtractBatch()` receives an inclusive `[batchStart, end]` window containing `batchSize+1` ledgers. The CLI then writes one oversized batch file set and never produces the separate tail batch implied by the flag contract.

## Trigger

Run:

`stellar-etl export_ledger_entry_changes -x ../stellar-core/src/stellar-core -c /etl/docker/stellar-core.cfg -s 49265302 -e 49265366 -b 64 -o /tmp/changes`

The requested range spans 65 ledgers. The documented behavior suggests a `64 + 1` split, but the current implementation can export a single `49265302-49265366-*` batch per resource instead.

## Target Code

- `internal/input/changes.go:StreamChanges:163-175` — inclusive batch extraction plus `batchStart+batchSize` initialization creates a `batchSize+1` first batch when `end == batchStart+batchSize`
- `cmd/export_ledger_entry_changes.go:exportTransformedData:295-376` — writes exactly one file set per emitted batch, so a merged batch suppresses the expected tail batch entirely
- `internal/utils/main.go:AddCoreFlags:271-277` — user-facing flag text says `batch-size` is the number of ledgers per batch
- `internal/input/changes_test.go:TestStreamChangesBatchNumbers:159-179` — current regression expectations already encode `1..65` as a single batch for `batchSize=64`, showing the off-by-one behavior is live

## Evidence

The test table in `changes_test.go` names the `1..65` case `"single"` and expects one batch `[1,65]` with `batchSize=64`, while the `"one extra"` `1..66` case expects `[1,64]` and `[65,66]`. That matches the current `StreamChanges()` math exactly and demonstrates that the implementation treats `batchStart+batchSize == end` as one inclusive batch of `batchSize+1` ledgers rather than a full batch plus a one-ledger tail.

## Anti-Evidence

Individual change rows inside the oversized batch are still derived from real ledgers, so this is not a field-level corruption like a wrong fee or balance value. The impact is structural: batch sharding, file-count expectations, and any downstream workflow keyed to the documented per-batch ledger count become wrong even though each exported row still looks valid.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (fail/029 noted the same pattern in passing but investigated a different hypothesis)
**Failed At**: reviewer

### Trace Summary

Traced `StreamChanges` (changes.go:162-178) with `start=49265302, end=49265366, batchSize=64`. Confirmed the mechanism: `batchEnd = min(49265366, 49265366) = 49265366`, the guard `49265366 < 49265366` is FALSE so no decrement, and `ExtractBatch(49265302, 49265366)` is called with 65 ledgers. The math is correct as stated. However, this behavior is explicitly encoded as the expected outcome in `TestStreamChangesBatchNumbers` — the test case named `"single"` verifies that range [1, 65] with batchSize=64 produces exactly one batch [{1, 65}]. This is working-as-designed, not a bug.

### Code Paths Examined

- `internal/input/changes.go:StreamChanges:162-178` — Confirmed the batchEnd computation and the conditional decrement. When `batchEnd == end`, no decrement occurs, producing an inclusive [start, end] batch of batchSize+1 ledgers.
- `internal/input/changes.go:extractBatch:82-158` — Each ledger within the batch gets fresh `changeCompactors` (line 102-105), so per-ledger compaction integrity is maintained regardless of batch size. No cross-ledger state leakage.
- `internal/input/changes_test.go:TestStreamChangesBatchNumbers:154-229` — Four test cases explicitly encode the batching behavior: `"single"` ([1,65] → one batch), `"one extra"` ([1,66] → two batches [1,64]+[65,66]), `"multiple"` ([1,128] → two batches), `"partial"` ([1,32] → one batch). The `"single"` case directly tests and accepts the batchSize+1 behavior.
- `cmd/export_ledger_entry_changes.go:exportTransformedData:295-377` — Uses `batch.BatchStart` and `batch.BatchEnd` for file naming. The file will correctly reflect the actual ledger range covered.
- `internal/utils/main.go:AddCoreFlags:271` — Flag text says "number of ledgers to export changes from in each batches" — this is ambiguous between "target" and "maximum".

### Why It Failed

The hypothesis treats explicitly tested and accepted behavior as a bug. The `TestStreamChangesBatchNumbers` test case `"single"` was deliberately written to verify that a range of batchSize+1 ledgers produces a single batch — the test name itself signals developer intent that this is one logical batch, not an oversight. The flag description ("number of ledgers to export changes from in each batches") is a target size, not a strict maximum. The implementation intentionally absorbs the +1 tail ledger into the current batch to avoid creating trivially small 1-ledger tail batches.

Furthermore, data correctness is unaffected:
1. **No data loss**: All ledgers are processed exactly once.
2. **No value corruption**: `extractBatch` creates fresh compactors per ledger (line 102), so per-ledger change compaction is identical regardless of batch boundaries.
3. **No duplication**: The `batchStart = min(batchEnd, end) + 1` advance (line 173) ensures non-overlapping batches.

The batch file naming at `exportTransformedData` line 309 uses the actual `start` and `end+1` values, so file names accurately reflect the ledger range covered. Downstream consumers parsing these filenames will see the correct range.

### Lesson Learned

When a test explicitly encodes edge-case behavior with a descriptive name (e.g., `"single"` for a batchSize+1 range), that is strong evidence of intentional design. A batch-size flag text saying "number of ledgers per batch" is ambiguous and does not establish a strict maximum contract. For this codebase, treating near-batchSize ranges as a single batch is the documented and tested behavior, not a bug.
