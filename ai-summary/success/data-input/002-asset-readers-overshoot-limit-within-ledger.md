# 002: asset readers overshoot `--limit` within a single ledger

**Date**: 2026-04-10
**Severity**: Medium
**Impact**: Operational correctness: `--limit` can return oversized asset exports
**Subsystem**: data-input
**Final review by**: gpt-5.4, high

## Summary

`export_assets --limit N` does not stop at `N` qualifying operations when the boundary is crossed inside a ledger. Both asset readers finish scanning the rest of the current ledger before checking the limit, so a request for `--limit 1` can still return dozens of payment/manage-sell operations and multiple distinct asset rows.

On mainnet ledger `30820015`, both production asset readers returned `56` qualifying operations with `limit=1`, and `export_assets` would still emit `16` unique asset rows after its normal deduplication step. The sibling operations reader honors the same limit exactly on the same ledger, showing this is an asset-reader-specific bug rather than a property of the underlying data source.

## Root Cause

`GetPaymentOperations()` and `GetPaymentOperationsHistoryArchive()` append qualifying operations inside nested transaction/operation loops, but they only test `len(assetSlice) >= limit` after the entire ledger has been processed. That means the first ledger that crosses the boundary is returned in full. `GetOperations()` uses inner-loop and post-append limit checks, so the inconsistent control flow is the actual cause of the overshoot.

## Reproduction

During normal operation, running `stellar-etl export_assets -s 30820015 -e 30820015 -l 1` succeeds but writes `16` rows instead of `1`. The same ledger, read through the production input package, returns `56` qualifying asset operations from both the datastore-backed reader and the history-archive reader despite `limit=1`.

## Affected Code

- `internal/input/assets.go:31-57` — appends every qualifying payment/manage-sell operation in the ledger before performing the only limit check
- `internal/input/assets_history_archive.go:21-47` — repeats the same outer-only limit enforcement in the history-archive reader
- `internal/input/operations.go:52-77` — sibling reader shows the intended inner-loop break behavior and stops exactly at the requested limit
- `cmd/export_assets.go:30-70` — passes the caller-visible `limit` directly into the asset readers, then only deduplicates repeated assets after the oversized read has already happened
- `internal/utils/main.go:250-254` — defines `--limit` as the maximum number of assets to export

## PoC

- **Target test file**: `internal/input/data_integrity_poc_test.go`
- **Test name**: `TestAssetReaderOvershootsLimitWithinLedger`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package input

import (
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
	"github.com/stellar/stellar-etl/v2/internal/utils"
)

func TestAssetReaderOvershootsLimitWithinLedger(t *testing.T) {
	const ledger = uint32(30820015)
	const limit = int64(1)

	env := utils.GetEnvironmentDetails(utils.CommonFlagValues{
		DatastorePath: "sdf-ledger-close-meta/v1/ledgers",
		BufferSize:    200,
		NumWorkers:    10,
		RetryLimit:    3,
		RetryWait:     5,
	})

	paymentOps, err := GetPaymentOperations(ledger, ledger, limit, env, false)
	if err != nil {
		t.Fatalf("GetPaymentOperations returned error: %v", err)
	}
	if len(paymentOps) <= int(limit) {
		t.Fatalf("expected GetPaymentOperations to overshoot limit=%d within ledger %d, got %d result(s)", limit, ledger, len(paymentOps))
	}

	uniqueAssets := map[int64]struct{}{}
	for _, paymentOp := range paymentOps {
		asset, err := transform.TransformAsset(
			paymentOp.Operation,
			paymentOp.OperationIndex,
			paymentOp.TransactionIndex,
			paymentOp.LedgerSeqNum,
			paymentOp.LedgerCloseMeta,
		)
		if err != nil {
			t.Fatalf("TransformAsset returned error: %v", err)
		}
		uniqueAssets[asset.AssetID] = struct{}{}
	}
	if len(uniqueAssets) <= int(limit) {
		t.Fatalf("expected overshoot to survive export_assets dedupe with limit=%d, got %d unique asset(s)", limit, len(uniqueAssets))
	}

	ops, err := GetOperations(ledger, ledger, limit, env, false)
	if err != nil {
		t.Fatalf("GetOperations returned error: %v", err)
	}
	if len(ops) != int(limit) {
		t.Fatalf("expected GetOperations to honor limit=%d on the same ledger, got %d result(s)", limit, len(ops))
	}

	historyArchiveOps, err := GetPaymentOperationsHistoryArchive(ledger, ledger, limit, env, true)
	if err != nil {
		t.Fatalf("GetPaymentOperationsHistoryArchive returned error: %v", err)
	}
	if len(historyArchiveOps) <= int(limit) {
		t.Fatalf("expected GetPaymentOperationsHistoryArchive to overshoot limit=%d within ledger %d, got %d result(s)", limit, ledger, len(historyArchiveOps))
	}

	t.Logf(
		"BUG CONFIRMED: asset readers returned %d datastore ops and %d history-archive ops with limit=%d on ledger %d; export_assets would emit %d unique asset(s)",
		len(paymentOps),
		len(historyArchiveOps),
		limit,
		ledger,
		len(uniqueAssets),
	)
}
```

## Expected vs Actual Behavior

- **Expected**: `export_assets --limit 1` should stop after the first qualifying asset operation and emit at most one asset row, matching the flag description and sibling reader behavior.
- **Actual**: both asset readers return the rest of the current ledger after the first hit, so `export_assets --limit 1` on ledger `30820015` emits `16` unique asset rows.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls both production asset readers with `limit=1` on a real ledger and compares them with `GetOperations`, which honors the same limit exactly.
2. Realistic preconditions: YES — the reproducer uses a historical mainnet ledger already referenced by the repo's own asset export fixtures, and the same oversized output is visible from the shipped CLI.
3. Bug vs by-design: BUG — `--limit` is documented as the maximum number of assets to export, and sibling readers implement exact stop conditions inside the inner loops; only the asset readers omit those checks.
4. Final severity: Medium — the exported rows are individually valid, but the command silently violates a caller-specified bound and can include extra rows from the same ledger.
5. In scope: YES — this is a concrete silent wrong-output path in production export code.
6. Test correctness: CORRECT — the test does not copy the buggy loop; it exercises the real readers, runs the real asset transform, and proves the extra rows survive normal asset deduplication.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Mirror the limit enforcement used by `GetOperations()` and `GetTrades()`: after each qualifying append in both asset readers, break once `len(assetSlice) >= limit && limit >= 0`, and retain the outer post-ledger break as a second guard. That preserves existing unlimited behavior for negative limits while making `--limit` exact for asset exports.
