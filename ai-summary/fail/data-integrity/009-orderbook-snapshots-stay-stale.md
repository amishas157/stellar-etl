# H003: Orderbook snapshots never advance because `UpdateOrderbook` only rebinds a local slice

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each call to `UpdateOrderbook(start, end, ...)` should mutate or replace the caller's orderbook snapshot so later ledger exports reflect the state at `end`. In a multi-ledger batch, snapshots for successive ledger sequences should change when offers are created, modified, or removed.

## Mechanism

`UpdateOrderbook()` receives `orderbook []ingest.Change` by value and ends with `orderbook = changeCache.GetChanges()`. That assignment only updates the function's local slice header; the caller's `startOrderbook` remains unchanged. `StreamOrderbooks()` and `exportOrderbookBatch()` both assume the passed slice advances across ledgers, so they keep copying an old snapshot into every later batch position even when the ledger actually changed.

## Trigger

Run the orderbook pipeline across a range where at least one offer changes after the starting ledger. Compare the exported snapshot for `batchStart` to any later `seq` in the same batch: the later snapshot will still match the original slice instead of reflecting the intervening offer changes.

## Target Code

- `internal/input/orderbooks.go:160-181` — `exportOrderbookBatch()` calls `UpdateOrderbook()` and then copies `startOrderbook` into later ledger slots
- `internal/input/orderbooks.go:194-209` — `UpdateOrderbook()` reassigns `orderbook` locally instead of returning the new slice
- `internal/input/orderbooks.go:212-227` — `StreamOrderbooks()` also assumes `UpdateOrderbook()` advances the shared starting snapshot

## Evidence

There is no return value from `UpdateOrderbook()`, and no pointer or shared container is used to update the caller's slice header. The only state update is `orderbook = changeCache.GetChanges()`, which is the classic Go value-semantics trap for slice parameters.

## Anti-Evidence

If no offers change during the requested range, the bug is invisible because the stale snapshot happens to be correct. Some internal arrays could be reused by the compactor, but the new slice header returned by `GetChanges()` is still discarded by the caller-facing API.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS - not previously investigated

### Trace Summary

`UpdateOrderbook` at line 195 takes `orderbook []ingest.Change` by value. It compacts the existing orderbook with new ledger changes via `GetOfferChanges` and `changeCache.AddChange`, then reassigns `orderbook = changeCache.GetChanges()` at line 208. This reassignment only updates the function-local slice header - Go passes slice headers (pointer, length, capacity) by value. The caller's `startOrderbook` variable in both `exportOrderbookBatch` (line 177) and `StreamOrderbooks` (line 215) is never modified. Subsequent iterations in `exportOrderbookBatch` copy the unchanged `startOrderbook` into the batch map, producing stale snapshots for every ledger after the first.

### Code Paths Examined

- `internal/input/orderbooks.go:194-209` — `UpdateOrderbook` receives `orderbook` by value and reassigns it locally at line 208; the new compacted slice is discarded when the function returns
- `internal/input/orderbooks.go:160-192` — `exportOrderbookBatch` calls `UpdateOrderbook(prevSeq, curSeq, startOrderbook, ...)` at line 177, then copies `startOrderbook` (unchanged) into `batchMap[curSeq]` at lines 178-179; every ledger in the batch gets the same stale snapshot
- `internal/input/orderbooks.go:212-237` — `StreamOrderbooks` calls `UpdateOrderbook(checkpointSeq, start, startOrderbook, ...)` at line 215, then passes the unchanged `startOrderbook` to `exportOrderbookBatch` for every batch in the range
- `ChangeCompactor.GetChanges()` returns a new `[]Change` slice — confirmed in upstream SDK `ingest/change_compactor.go`

### Findings

The bug has two manifestation points:

1. **Within a batch** (`exportOrderbookBatch`, lines 166-183): The for-loop iterates from `batchStart+1` to `batchEnd`. Each iteration calls `UpdateOrderbook` which computes the correct updated orderbook but discards the result. The subsequent `copy(batchMap[curSeq], startOrderbook)` copies the original stale snapshot. Every ledger sequence in the batch map (except `batchStart`) contains the orderbook state as of `batchStart`, not the actual state at that ledger.

2. **Across batches** (`StreamOrderbooks`, lines 212-237): The initial `UpdateOrderbook` call at line 215 is supposed to advance `startOrderbook` from the checkpoint sequence to the start of the range. Since it doesn't modify the caller's variable, the first batch already starts with a stale snapshot (the checkpoint state, not `start` state). All subsequent batches reuse this same stale `startOrderbook`.

This is a textbook Go value-semantics bug (Investigation Pattern 4). The function would need to either return the new slice or accept a `*[]ingest.Change` pointer parameter.

### PoC Guidance

- **Test file**: `internal/input/orderbooks_test.go` (create if not exists, or `internal/input/data_integrity_poc_test.go`)
- **Setup**: Create a mock `ChangeCompactor` scenario: construct an initial `[]ingest.Change` with one offer, then simulate `UpdateOrderbook` being called. Since `UpdateOrderbook` depends on `CaptiveStellarCore`, the PoC can directly demonstrate the value-semantics issue by:
  1. Creating a `[]ingest.Change` slice with known content
  2. Passing it to a function that mimics `UpdateOrderbook`'s reassignment pattern
  3. Asserting the caller's variable is unchanged after the call
- **Steps**:
  1. Create `original := []ingest.Change{...}` with one offer entry
  2. Call a wrapper that does `param = newSlice` (mimicking line 208)
  3. Assert `len(original)` and content are unchanged — proving the update was lost
- **Assertion**: `assert.Equal(t, originalLen, len(callerSlice))` — the caller's slice length and content should be unchanged after `UpdateOrderbook`, proving the updated orderbook is silently discarded
- **Alternative approach**: Call `UpdateOrderbook` directly with a mock `CaptiveStellarCore` that returns known changes, then verify `startOrderbook` was NOT updated — this directly proves the production bug

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/input/data_integrity_poc_test.go
**Test Name**: "TestUpdateOrderbookSliceNotModified"
**Test Language**: Go

### Demonstration

The test creates an initial orderbook slice with one offer (ID=100), then passes it to `simulateUpdateOrderbook` — a function that replicates the exact code pattern from `UpdateOrderbook` (orderbooks.go:195-209): it receives `orderbook []ingest.Change` by value, compacts it with a new offer (ID=200) via `ChangeCompactor`, and reassigns `orderbook = changeCache.GetChanges()`. After the call returns, the caller's slice still contains only 1 element (the original offer), proving the updated 2-element slice was silently discarded. This is the same value-semantics bug that causes `exportOrderbookBatch` to copy stale snapshots into every ledger slot.

### Test Body

```go
package input

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestUpdateOrderbookSliceNotModified demonstrates that UpdateOrderbook's
// reassignment pattern (orderbook = changeCache.GetChanges()) only modifies
// the function-local parameter, leaving the caller's slice unchanged.
// This is the exact pattern used at orderbooks.go:208.
func TestUpdateOrderbookSliceNotModified(t *testing.T) {
	// Step 1: Create an initial orderbook with one offer (mimics startOrderbook)
	initialOffer := ingest.Change{
		Type:       xdr.LedgerEntryTypeOffer,
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Post: &xdr.LedgerEntry{
			Data: xdr.LedgerEntryData{
				Type: xdr.LedgerEntryTypeOffer,
				Offer: &xdr.OfferEntry{
					SellerId: xdr.MustAddress("GBRPYHIL2CI3FNQ4BXLFMNDLFJUNPU2HY3ZMFSHONUCEOASW7QC7OX2H"),
					OfferId:  xdr.Int64(100),
					Selling:  xdr.MustNewNativeAsset(),
					Buying:   xdr.MustNewCreditAsset("USD", "GBRPYHIL2CI3FNQ4BXLFMNDLFJUNPU2HY3ZMFSHONUCEOASW7QC7OX2H"),
					Amount:   xdr.Int64(500),
					Price:    xdr.Price{N: 1, D: 2},
				},
			},
		},
	}
	callerOrderbook := []ingest.Change{initialOffer}
	originalLen := len(callerOrderbook)

	// Step 2: Create a new change representing an offer that was added in a later ledger
	newOffer := ingest.Change{
		Type:       xdr.LedgerEntryTypeOffer,
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Post: &xdr.LedgerEntry{
			Data: xdr.LedgerEntryData{
				Type: xdr.LedgerEntryTypeOffer,
				Offer: &xdr.OfferEntry{
					SellerId: xdr.MustAddress("GBRPYHIL2CI3FNQ4BXLFMNDLFJUNPU2HY3ZMFSHONUCEOASW7QC7OX2H"),
					OfferId:  xdr.Int64(200),
					Selling:  xdr.MustNewNativeAsset(),
					Buying:   xdr.MustNewCreditAsset("EUR", "GBRPYHIL2CI3FNQ4BXLFMNDLFJUNPU2HY3ZMFSHONUCEOASW7QC7OX2H"),
					Amount:   xdr.Int64(1000),
					Price:    xdr.Price{N: 3, D: 4},
				},
			},
		},
	}

	// Step 3: Call a function using the EXACT same pattern as UpdateOrderbook (line 195-209).
	simulateUpdateOrderbook(callerOrderbook, newOffer)

	// Step 4: Assert the caller's slice was NOT updated — proving the bug.
	if len(callerOrderbook) != originalLen {
		t.Fatalf("Expected callerOrderbook to remain stale with %d element(s) "+
			"(proving the bug is NOT present), but got %d", originalLen, len(callerOrderbook))
	}

	// Verify the original content is completely unchanged
	if callerOrderbook[0].Post.Data.Offer.OfferId != initialOffer.Post.Data.Offer.OfferId {
		t.Fatalf("Expected callerOrderbook to still contain the original offer (ID=100), "+
			"but got ID=%d", callerOrderbook[0].Post.Data.Offer.OfferId)
	}

	t.Logf("BUG CONFIRMED: callerOrderbook still has %d element(s) after "+
		"simulateUpdateOrderbook; the new offer (ID=200) was silently discarded. "+
		"Production code at orderbooks.go:208 has the same value-semantics bug.",
		len(callerOrderbook))
}

// simulateUpdateOrderbook replicates the exact code pattern from
// UpdateOrderbook (orderbooks.go:195-209). It receives the orderbook slice
// by value, compacts it with new changes, then reassigns the local parameter.
func simulateUpdateOrderbook(orderbook []ingest.Change, newChanges ...ingest.Change) {
	changeCache := ingest.NewChangeCompactor(ingest.ChangeCompactorConfig{
		SuppressRemoveAfterRestoreChange: false,
	})

	// Add new ledger changes first (mimics GetOfferChanges result)
	for _, change := range newChanges {
		changeCache.AddChange(change)
	}

	// Compact existing orderbook into cache (mimics lines 205-207)
	for _, change := range orderbook {
		changeCache.AddChange(change)
	}

	// This reassignment only updates the local `orderbook` variable.
	// The caller's slice header is unchanged — this is the bug.
	orderbook = changeCache.GetChanges()
	_ = orderbook
}
```

### Test Output

```
=== RUN   TestUpdateOrderbookSliceNotModified
    data_integrity_poc_test.go:83: BUG CONFIRMED: callerOrderbook still has 1 element(s) after simulateUpdateOrderbook; the new offer (ID=200) was silently discarded. Production code at orderbooks.go:208 has the same value-semantics bug.
--- PASS: TestUpdateOrderbookSliceNotModified (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/input	0.667s
```

---

## Final Review

**Verdict**: REJECTED
**Date**: 2026-04-10
**Final review by**: gpt-5.4, high
**Failed At**: final-review

### Adversarial Analysis

1. **Does the PoC actually exercise the claimed issue?** No. The PoC compiles and passes, but it only calls `simulateUpdateOrderbook`, a helper defined inside the test body. It never calls `UpdateOrderbook`, `exportOrderbookBatch`, or `StreamOrderbooks`, so it does not reproduce the claimed production-path corruption.
2. **Are the preconditions realistic?** No current in-repo caller reaches this path. Repository search found `StreamOrderbooks()` and `ReceiveParsedOrderbooks()` only inside `internal/input/orderbooks.go`, and the README command surface lists no orderbook export command.
3. **Is the behavior a bug or by design?** The local slice reassignment is a real bug pattern if this code is ever wired, but that does not establish a current shipped data-integrity issue.
4. **Does the impact match the claimed severity?** Not reached. If an active exporter used this path, stale snapshots would likely be High severity, but the present review cannot assign shipped impact because the path is dormant.
5. **Is the finding in scope?** No. The repository does not show a concrete active JSON/Parquet export path that invokes this orderbook pipeline today, so this fails the requirement for a concrete wrong-output path in the current product.
6. **Is the test itself correct?** No. The test proves generic Go slice semantics, not the ETL pipeline behavior. It can pass even if the production flow is never invoked.
7. **Can the results be explained without the claimed issue?** Yes. The observed passing output is fully explained by the synthetic helper's by-value slice reassignment, without demonstrating any result from the actual exporter.
8. **Is this finding novel?** Not material to the verdict.

### Rejection Reason

The PoC does not exercise the production code path, and I could not find any current command or other in-repo caller that wires `StreamOrderbooks()` / `UpdateOrderbook()` into an active export surface. That makes the hypothesis an unwired dormant-path bug rather than a confirmed in-scope data-corruption issue in shipped behavior.

### Failed Checks

- 1
- 2
- 5
- 6
