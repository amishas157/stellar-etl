# 005: Single-ledger change exports produce no batches

**Date**: 2026-04-11
**Severity**: High
**Impact**: Empty results from broken control flow
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`StreamChanges()` silently drops valid one-ledger ranges. When `start == end`, its batch loop never executes, so `export_ledger_entry_changes` can exit successfully without ever receiving a `ChangeBatch`, producing an empty export for a ledger that may contain real entry changes.

## Root Cause

`StreamChanges()` initializes `batchEnd` to `min(start+batchSize, end)` and then iterates only while `batchStart < batchEnd`. For a one-ledger request, `batchStart` and `batchEnd` are equal before the first iteration, so the function closes `changeChannel` immediately and signals completion without calling `ExtractBatch`.

## Reproduction

During normal operation, running `stellar-etl export_ledger_entry_changes --start-ledger L --end-ledger L ...` reaches `input.StreamChanges()` with equal `start` and `end` values. That is a valid range per `ValidateLedgerRange()`, upstream `ledgerbackend.SingleLedgerRange()` encodes the same `[L,L]` convention, but the loop guard skips the only batch, the command observes channel completion, and the run finishes with no exported rows instead of one batch covering ledger `L`.

## Affected Code

- `internal/input/changes.go:162-177` — `StreamChanges` computes equal `batchStart`/`batchEnd` for one-ledger ranges and skips the loop entirely
- `cmd/export_ledger_entry_changes.go:76-86` — the command treats the closed channel as normal completion and exits cleanly when no batch is ever sent
- `cmd/export_ledger_entry_changes.go:306-310` — filename generation explicitly treats `BatchEnd` as inclusive, confirming that a single-ledger batch should be representable
- `internal/utils/main.go:736-758` — ledger-range validation allows `start == end`, so this path is reachable in normal CLI usage
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/ingest/ledgerbackend/range.go:71-79` — upstream range helpers define a bounded single-ledger range as `from == to`

## PoC

- **Target test file**: `internal/input/data_integrity_poc_test.go`
- **Test name**: `TestSingleLedgerStreamChangesProducesNoBatch`
- **Test language**: `go`
- **How to run**: `cd <repo-root> && go build ./... && go test ./internal/input/... -run TestSingleLedgerStreamChangesProducesNoBatch -v`

### Test Body

```go
package input

import (
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/utils"
)

func TestSingleLedgerStreamChangesProducesNoBatch(t *testing.T) {
	oldExtractBatch := ExtractBatch
	ExtractBatch = mockExtractBatch
	defer func() {
		ExtractBatch = oldExtractBatch
	}()

	changeChan := make(chan ChangeBatch, 4)
	closeChan := make(chan int, 1)
	env := utils.EnvironmentDetails{}
	logger := utils.NewEtlLogger()

	go StreamChanges(nil, 5, 5, 64, changeChan, closeChan, env, logger)

	var batches []ChangeBatch
	for batch := range changeChan {
		batches = append(batches, batch)
	}

	if len(batches) != 1 {
		t.Fatalf("expected one batch for single-ledger range [5,5], got %d", len(batches))
	}

	if batches[0].BatchStart != 5 || batches[0].BatchEnd != 5 {
		t.Fatalf("expected batch [5,5], got [%d,%d]", batches[0].BatchStart, batches[0].BatchEnd)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: a valid one-ledger export should emit exactly one `ChangeBatch` covering that ledger, even if the batch contains zero or many compacted changes.
- **Actual**: `StreamChanges()` emits zero batches when `start == end`, so callers finish successfully with an empty result set.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the production `StreamChanges()` function with a valid one-ledger range and fails because the batching loop never invokes `ExtractBatch`.
2. Realistic preconditions: YES — `export_ledger_entry_changes` passes CLI ledger bounds directly into `StreamChanges()`, and equal start/end values are accepted by validation and upstream range helpers.
3. Bug vs by-design: BUG — the export command, upstream range API, and filename logic all model bounded ranges as inclusive, so treating `[L,L]` as empty is inconsistent control flow rather than an intentional convention.
4. Final severity: High — this is silent structural data loss in a normal export path, not a direct monetary miscalculation.
5. In scope: YES — it is a concrete production code path that produces wrong-but-plausible empty output.
6. Test correctness: CORRECT — the test asserts the intended one-batch behavior and fails on current code for the claimed reason; mocking `ExtractBatch` is appropriate because the bug occurs before any backend I/O.
7. Alternative explanations: NONE — the missing batch is fully explained by the `batchStart < batchEnd` guard.
8. Novelty: NOT ASSESSED HERE — duplicate handling is owned by the orchestrator.

## Suggested Fix

Change the `StreamChanges()` loop guard from `batchStart < batchEnd` to `batchStart <= batchEnd` so one-ledger ranges execute exactly one batch. Keep the existing non-overlap update logic unchanged; with the inclusive guard, the current `batchStart = batchEnd + 1` step still terminates cleanly after the single batch.
