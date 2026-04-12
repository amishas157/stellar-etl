# 017: `export_assets --limit 0` still emits first-ledger asset rows

**Date**: 2026-04-12
**Severity**: Medium
**Impact**: Operational correctness / zero-limit contract violation
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`export_assets` passes the user-visible `--limit` straight into `input.GetPaymentOperations()`, but that reader appends every qualifying payment or manage-sell-offer operation from the current ledger before checking whether the limit has been exhausted. When `limit=0`, the first in-range ledger is still fully scanned, so an explicitly empty request can return plausible non-empty asset output.

The original PoC needed correction because its target test file did not exist and it replayed the loop logic instead of calling production code. A fixed PoC that calls the real datastore-backed reader on mainnet ledger `30820015` shows `limit=0` returning `56` candidate operations and `16` distinct asset rows, matching the same single-ledger operation count as an unbounded read.

## Root Cause

`GetPaymentOperations()` and `GetPaymentOperationsHistoryArchive()` enforce `limit` only after each ledger-wide append phase completes. `cmd/export_assets.go` then transforms and writes every returned item, with no zero-limit short circuit and no row-level cap after deduplication.

## Reproduction

During normal operation, any first in-range ledger that contains qualifying `Payment` or `ManageSellOffer` operations can trigger the bug. With `--start-ledger 30820015 --end-ledger 30820015 --limit 0`, the datastore-backed reader still returns all qualifying operations from that ledger, and `TransformAsset()` turns them into non-empty exported asset rows.

## Affected Code

- `internal/utils/main.go:AddArchiveFlags:250-254` — documents `--limit` as the maximum number of exported assets
- `internal/input/assets.go:GetPaymentOperations:21-60` — appends qualifying operations before the post-ledger limit check
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:13-50` — repeats the same post-ledger-only limit enforcement
- `cmd/export_assets.go:17-82` — transforms and writes every returned asset-discovery operation without a second cap

## PoC

- **Target test file**: `internal/input/data_integrity_poc_test.go`
- **Test name**: `TestAssetReaderZeroLimitEmitsItems`
- **Test language**: `go`
- **How to run**: Create the target test file with the body below, then run `go test ./internal/input/... -run TestAssetReaderZeroLimitEmitsItems -v`.

### Test Body

```go
package input

import (
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
	"github.com/stellar/stellar-etl/v2/internal/utils"
)

func TestAssetReaderZeroLimitEmitsItems(t *testing.T) {
	env := utils.GetEnvironmentDetails(utils.CommonFlagValues{
		DatastorePath: "sdf-ledger-close-meta/v1/ledgers",
		BufferSize:    200,
		NumWorkers:    10,
		RetryLimit:    3,
		RetryWait:     5,
	})

	const ledger uint32 = 30820015

	zeroLimitOps, err := GetPaymentOperations(ledger, ledger, 0, env, false)
	if err != nil {
		t.Fatalf("GetPaymentOperations(limit=0) returned error: %v", err)
	}

	if len(zeroLimitOps) == 0 {
		t.Fatalf("expected limit=0 to still return first-ledger payment operations on the vulnerable path, got 0")
	}

	unboundedOps, err := GetPaymentOperations(ledger, ledger, -1, env, false)
	if err != nil {
		t.Fatalf("GetPaymentOperations(limit=-1) returned error: %v", err)
	}

	if len(zeroLimitOps) != len(unboundedOps) {
		t.Fatalf("expected single-ledger limit=0 to return the same operations as unbounded read, got %d vs %d", len(zeroLimitOps), len(unboundedOps))
	}

	distinctAssets := map[int64]struct{}{}
	for _, op := range zeroLimitOps {
		asset, err := transform.TransformAsset(op.Operation, op.OperationIndex, op.TransactionIndex, op.LedgerSeqNum, op.LedgerCloseMeta)
		if err != nil {
			t.Fatalf("TransformAsset returned error for returned operation: %v", err)
		}
		distinctAssets[asset.AssetID] = struct{}{}
	}

	if len(distinctAssets) == 0 {
		t.Fatalf("expected limit=0 reader output to yield at least one exported asset row, got 0")
	}

	t.Logf("BUG CONFIRMED: limit=0 returned %d candidate ops and %d distinct asset rows from ledger %d", len(zeroLimitOps), len(distinctAssets), ledger)
}
```

## Expected vs Actual Behavior

- **Expected**: `export_assets --limit 0` should emit zero asset rows.
- **Actual**: the first in-range ledger is still scanned and exported, yielding non-empty asset output even though the caller requested zero.

## Adversarial Review

1. Exercises claimed bug: YES — the fixed PoC calls the production datastore-backed asset reader and then the production `TransformAsset()` path, proving non-empty exportable rows for `limit=0`.
2. Realistic preconditions: YES — `--limit 0` is a valid CLI value, and many real ledgers, including `30820015`, contain qualifying asset-discovery operations.
3. Bug vs by-design: BUG — the public flag help says `limit` is the maximum number of exported assets, and zero cannot reasonably mean “export the first ledger anyway.”
4. Final severity: Medium — the row contents are real chain data, but the exporter silently violates an explicit output-bound contract and can return a non-empty file for an explicitly empty request.
5. In scope: YES — this is a concrete data-export correctness bug producing silent wrong output.
6. Test correctness: CORRECT — the final PoC uses the actual production reader instead of replaying logic, compares zero-limit behavior to an unbounded single-ledger read, and confirms the returned items become emitted asset rows.
7. Alternative explanations: NONE — the only reason rows appear is that the limit check runs after the ledger-wide append phase.
8. Novelty: NOVEL

## Suggested Fix

Short-circuit `limit == 0` before scanning any ledgers, and move asset-reader limit enforcement into the inner transaction/operation loops so no qualifying operations are appended once the budget is exhausted. If the intended contract is truly “maximum number of exported assets” rather than “maximum number of discovery operations,” add a second cap after `TransformAsset()` and deduplication in `cmd/export_assets.go`.
