# 003: Single-ledger change exports produce no batches

**Date**: 2026-04-11
**Severity**: High
**Impact**: Empty results from broken control flow
**Subsystem**: data-input
**Final review by**: gpt-5.4, high

## Summary

`StreamChanges()` silently drops valid one-ledger ranges. When `start == end`, its batch loop never executes, so `export_ledger_entry_changes` can exit successfully without ever receiving a `ChangeBatch`, producing an empty export for a ledger that may contain real entry changes.

## Root Cause

`StreamChanges()` initializes `batchEnd` to `min(start+batchSize, end)` and then iterates only while `batchStart < batchEnd`. For a one-ledger request, `batchStart` and `batchEnd` are equal before the first iteration, so the function closes `changeChannel` immediately and signals completion without calling `ExtractBatch`.

## Reproduction

During normal operation, running `stellar-etl export_ledger_entry_changes --start-ledger L --end-ledger L ...` reaches `input.StreamChanges()` with equal `start` and `end` values. That is a valid range per `ValidateLedgerRange()`, but the loop guard skips the only batch, the command observes channel completion, and the run finishes with no exported rows instead of one batch covering ledger `L`.

## Affected Code

- `internal/input/changes.go:162-177` — `StreamChanges` computes equal `batchStart`/`batchEnd` for one-ledger ranges and skips the loop entirely
- `cmd/export_ledger_entry_changes.go:76-86` — the command treats the closed channel as normal completion and exits cleanly when no batch is ever sent
- `internal/utils/main.go:736-758` — ledger-range validation allows `start == end`, so this path is reachable in normal CLI usage
- `internal/input/changes_test.go:142-229` — existing batching tests miss the equality case and therefore do not catch the bug

## PoC

- **Target test file**: `internal/input/data_integrity_poc_test.go`
- **Test name**: `TestSingleLedgerStreamChangesProducesNoBatches`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package input

import (
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/utils"
	"github.com/stretchr/testify/require"
)

func TestSingleLedgerStreamChangesProducesNoBatches(t *testing.T) {
	originalExtractBatch := ExtractBatch
	t.Cleanup(func() {
		ExtractBatch = originalExtractBatch
	})
	ExtractBatch = mockExtractBatch

	changeChan := make(chan ChangeBatch, 1)
	closeChan := make(chan int, 1)
	logger := utils.NewEtlLogger()

	go StreamChanges(nil, 50, 50, 64, changeChan, closeChan, utils.EnvironmentDetails{}, logger)

	var batches []ChangeBatch
	for batch := range changeChan {
		batches = append(batches, batch)
	}
	<-closeChan

	require.Len(t, batches, 1, "single-ledger exports should emit exactly one batch")
	require.Equal(t, uint32(50), batches[0].BatchStart)
	require.Equal(t, uint32(50), batches[0].BatchEnd)
}
```

## Expected vs Actual Behavior

- **Expected**: a valid one-ledger export should emit exactly one `ChangeBatch` covering that ledger, even if the batch contains zero or many compacted changes.
- **Actual**: `StreamChanges()` emits zero batches when `start == end`, so callers finish successfully with an empty result set.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the production `StreamChanges()` path with a valid one-ledger range and observes that no batch is emitted.
2. Realistic preconditions: YES — `export_ledger_entry_changes` passes CLI ledger bounds directly into `StreamChanges()`, and equal start/end values are accepted by validation.
3. Bug vs by-design: BUG — the existing batching tests and `extractBatch()` both treat batch endpoints as inclusive ranges, so dropping the only ledger is inconsistent control flow rather than an intentional empty-range convention.
4. Final severity: High — this is silent structural data loss in a normal export path, not a direct monetary miscalculation.
5. In scope: YES — it is a concrete production code path that produces wrong-but-plausible empty output.
6. Test correctness: CORRECT — unlike the original PoC, the final test asserts the intended behavior and fails on current code; changing the loop guard to `<=` makes both this test and the preexisting batch-range tests pass.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Change the `StreamChanges()` loop guard from `batchStart < batchEnd` to `batchStart <= batchEnd` so one-ledger ranges execute exactly one batch. Keep the existing non-overlap update logic unchanged; with the inclusive guard, the current `batchStart = batchEnd + 1` step still terminates cleanly after the single batch.
