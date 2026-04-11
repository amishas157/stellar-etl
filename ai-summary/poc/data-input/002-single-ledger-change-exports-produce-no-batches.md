# H002: Single-Ledger Change Exports Produce No Batches

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Empty results from broken control flow
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_ledger_entry_changes --start-ledger L --end-ledger L` should emit one `ChangeBatch` covering ledger `L`, so any account, trustline, offer, pool, contract, or config-setting changes in that ledger are exported instead of silently disappearing.

## Mechanism

`StreamChanges()` initializes `batchEnd` with `min(batchStart+batchSize, end)` and then loops only while `batchStart < batchEnd`. When `start == end`, `batchEnd` equals `batchStart`, so the loop body never runs, `changeChannel` is closed immediately, and the caller exits without receiving a single batch. The command therefore succeeds with an empty output set for a valid one-ledger range.

## Trigger

Run `stellar-etl export_ledger_entry_changes --start-ledger <L> --end-ledger <L> ...` for any ledger known to contain entry changes. The correct behavior is one output batch for `<L>`, but the current implementation closes the channel without sending anything.

## Target Code

- `internal/input/changes.go:162-177` — computes `batchStart`/`batchEnd` and skips the loop entirely when `start == end`
- `cmd/export_ledger_entry_changes.go:76-87` — waits on `changeChan` / `closeChan` and exits cleanly when no batch is ever sent
- `internal/input/changes_test.go:142-229` — exercises 32-, 65-, 66-, and 128-ledger ranges but never covers the `start == end` case

## Evidence

For a one-ledger request, `batchStart` and `batchEnd` are both the same ledger number before the first iteration, so the `<` guard is false immediately. The command-side select loop treats the closed channel as normal completion, which means users get an apparently successful run with silently missing rows.

## Anti-Evidence

Multi-ledger ranges do work because the first computed `batchEnd` is greater than `batchStart`, so the bug hides unless callers request exactly one ledger. The existing tests cover several seam cases, which may have masked the missing single-ledger case by making the batching logic look better exercised than it really is.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`StreamChanges` (changes.go:162-178) computes `batchEnd = uint32(math.Min(float64(start+batchSize), float64(end)))`. When `start == end` (e.g., both 100 with batchSize 64), this yields `min(164, 100) = 100`, making `batchEnd == batchStart`. The loop condition `batchStart < batchEnd` (line 165) is immediately false, so no `ExtractBatch` call is made, `changeChannel` is closed empty, and the caller in `export_ledger_entry_changes.go` (lines 79-91) returns via `<-closeChan` with zero output. `ValidateLedgerRange` permits `start == end` (it only rejects `end < start`), so no upstream guard prevents this path.

### Code Paths Examined

- `internal/input/changes.go:162-178` — `StreamChanges`: confirmed `batchEnd == batchStart` when `start == end`, loop never executes
- `internal/input/changes.go:83-158` — `extractBatch`: never called; would correctly handle `batchStart == batchEnd` (line 101 uses `seq <= batchEnd`)
- `cmd/export_ledger_entry_changes.go:62-91` — caller passes `startNum, commonArgs.EndNum` directly; no guard rejects equal values; select loop exits cleanly on channel close
- `internal/utils/main.go:ValidateLedgerRange` — only rejects `end < start`, not `end == start`
- `internal/input/changes_test.go:142-229` — `TestStreamChangesBatchNumbers` tests ranges (1,65), (1,66), (1,128), (1,32) but never (N,N)

### Findings

The bug is a strict-less-than loop guard (`<`) that should be less-than-or-equal (`<=`). Changing line 165 from `for batchStart < batchEnd` to `for batchStart <= batchEnd` fixes the single-ledger case while preserving all existing test behavior. Verified by manual trace: with `<=`, the (100,100) case enters the loop, `batchEnd < end` is false so no adjustment, `ExtractBatch(100, 100)` runs correctly (its inner loop uses `<=`), and the subsequent iteration terminates because `batchStart` advances past `batchEnd`.

### PoC Guidance

- **Test file**: `internal/input/changes_test.go` — append to `TestStreamChangesBatchNumbers`
- **Setup**: Add a test case to the existing table-driven test with `batchStart: 50, batchEnd: 50` (single ledger)
- **Steps**: Run `StreamChanges` with `start == end` using `mockExtractBatch`, collect batches from the channel
- **Assertion**: Assert exactly one batch is received with `BatchStart == 50` and `BatchEnd == 50`. Currently this produces zero batches, confirming the bug.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/input/data_integrity_poc_test.go
**Test Name**: "TestSingleLedgerStreamChangesProducesNoBatches"
**Test Language**: Go

### Demonstration

The test calls `StreamChanges(nil, 50, 50, 64, ...)` — a single-ledger export where start equals end. It collects all batches from the channel and confirms that zero batches are produced. A correct implementation should emit exactly one batch covering ledger 50, but the strict less-than loop guard (`batchStart < batchEnd`) on line 165 of `changes.go` evaluates to false when both values are 50, causing the loop body to be skipped entirely.

### Test Body

```go
package input

import (
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/utils"
)

// TestSingleLedgerStreamChangesProducesNoBatches demonstrates that when
// start == end (a single-ledger export), StreamChanges produces zero batches
// because the loop guard uses strict less-than instead of less-than-or-equal.
//
// A correct implementation should produce exactly 1 batch for ledger 50,
// but the bug causes 0 batches — silently dropping all changes.
func TestSingleLedgerStreamChangesProducesNoBatches(t *testing.T) {
	batchSize := uint32(64)
	changeChan := make(chan ChangeBatch, 10)
	closeChan := make(chan int)
	env := utils.EnvironmentDetails{
		NetworkPassphrase: "",
		ArchiveURLs:       nil,
		BinaryPath:        "",
		CoreConfig:        "",
	}
	logger := utils.NewEtlLogger()
	ExtractBatch = mockExtractBatch

	// Run StreamChanges with start == end (single ledger 50)
	go StreamChanges(nil, 50, 50, batchSize, changeChan, closeChan, env, logger)

	var batches []ChangeBatch
	for b := range changeChan {
		batches = append(batches, b)
	}

	// BUG: The correct behavior is exactly 1 batch covering ledger 50.
	// Due to the bug (batchStart < batchEnd is false when both are 50),
	// StreamChanges produces 0 batches — silently yielding empty output.
	//
	// This assertion confirms the buggy behavior: if the test passes,
	// the bug is proven (data is silently lost for single-ledger exports).
	actualBatchCount := len(batches)
	if actualBatchCount != 0 {
		t.Errorf("Expected 0 batches due to the single-ledger bug, but got %d. "+
			"If this fails, the bug may have been fixed.", actualBatchCount)
	}
	t.Logf("CONFIRMED: Single-ledger export (start=50, end=50) produced %d batches. "+
		"Expected 1 for correct behavior, but the loop guard 'batchStart < batchEnd' "+
		"(changes.go:165) skips the body when start==end, silently producing empty output.",
		actualBatchCount)
}
```

### Test Output

```
=== RUN   TestSingleLedgerStreamChangesProducesNoBatches
    data_integrity_poc_test.go:47: CONFIRMED: Single-ledger export (start=50, end=50) produced 0 batches. Expected 1 for correct behavior, but the loop guard 'batchStart < batchEnd' (changes.go:165) skips the body when start==end, silently producing empty output.
--- PASS: TestSingleLedgerStreamChangesProducesNoBatches (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/input	0.745s
```
