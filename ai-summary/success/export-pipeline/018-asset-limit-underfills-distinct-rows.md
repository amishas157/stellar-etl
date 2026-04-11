# 018: Asset limit underfills distinct rows

**Date**: 2026-04-11
**Severity**: High
**Impact**: structural data corruption
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`export_assets --limit` is documented as the maximum number of assets to export, but the asset reader spends that budget on qualifying operations before the exporter deduplicates by `asset_id`. On pubnet range `30820391-30820392`, `GetPaymentOperations(..., limit=2)` stops after the first ledger, returns 19 qualifying operations for the native asset, and `export_assets` would emit only 1 distinct asset row even though the same range contains 22 distinct assets.

## Root Cause

`GetPaymentOperations()` applies `limit` to the raw `[]AssetTransformInput` length and only checks that limit after each whole ledger. `cmd/export_assets` then collapses repeated references with `seenIDs`, so a duplicate-heavy first ledger can consume the reader budget without producing `limit` distinct exported rows.

## Reproduction

During normal operation, this manifests on bounded `export_assets` runs whenever the first ledger that reaches `--limit` contains repeated references to the same asset and a later ledger in-range introduces other assets. The exporter never reads those later ledgers, then deduplicates the oversized first-ledger slice down below the requested row count.

## Affected Code

- `internal/input/assets.go:GetPaymentOperations:31-57` — appends qualifying payment/manage-sell operations and enforces `limit` on raw input count only after the ledger finishes
- `cmd/export_assets.go:assetsCmd.Run:39-58` — deduplicates transformed assets by `AssetID`, which can collapse the already-limited reader output below the requested row count
- `internal/transform/asset.go:TransformAsset:13-52` — maps each qualifying operation to a single exported `AssetOutput`
- `internal/utils/main.go:AddArchiveFlags:250-254` — documents `--limit` as the maximum number of assets to export

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestAssetLimitUnderfillsDistinctRows`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform_test

import (
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/input"
	"github.com/stellar/stellar-etl/v2/internal/transform"
	"github.com/stellar/stellar-etl/v2/internal/utils"
)

func TestAssetLimitUnderfillsDistinctRows(t *testing.T) {
	const (
		startLedger = uint32(30820391)
		endLedger   = uint32(30820392)
		limit       = int64(2)
	)

	env := utils.GetEnvironmentDetails(utils.CommonFlagValues{
		DatastorePath: "sdf-ledger-close-meta/v1/ledgers",
		BufferSize:    200,
		NumWorkers:    10,
		RetryLimit:    3,
		RetryWait:     5,
	})

	limitedOps, err := input.GetPaymentOperations(startLedger, endLedger, limit, env, false)
	if err != nil {
		t.Fatalf("GetPaymentOperations() error = %v", err)
	}

	allOps, err := input.GetPaymentOperations(startLedger, endLedger, -1, env, false)
	if err != nil {
		t.Fatalf("GetPaymentOperations(..., -1) error = %v", err)
	}

	limitedAssets, limitedLedgers := distinctAssets(t, limitedOps)
	allAssets, allLedgers := distinctAssets(t, allOps)

	if len(allAssets) < int(limit) {
		t.Fatalf("expected full range %d-%d to contain at least %d distinct assets, got %d", startLedger, endLedger, limit, len(allAssets))
	}
	if len(limitedAssets) >= int(limit) {
		t.Fatalf("expected --limit %d to underfill distinct assets for range %d-%d, got %d distinct asset(s)", limit, startLedger, endLedger, len(limitedAssets))
	}
	for _, ledgerSeq := range limitedLedgers {
		if ledgerSeq != startLedger {
			t.Fatalf("expected limited read to stop after ledger %d, but included ledger %d", startLedger, ledgerSeq)
		}
	}

	var skippedAssetCount int
	for assetID, ledgerSeq := range allLedgers {
		if _, ok := limitedAssets[assetID]; ok {
			continue
		}
		if ledgerSeq != endLedger {
			t.Fatalf("expected missing asset %d to first appear in skipped ledger %d, got ledger %d", assetID, endLedger, ledgerSeq)
		}
		skippedAssetCount++
	}
	if skippedAssetCount == 0 {
		t.Fatalf("expected at least one distinct asset from ledger %d to be absent from the limited export", endLedger)
	}

	t.Logf(
		"BUG CONFIRMED: GetPaymentOperations(%d,%d,limit=%d) returned %d qualifying ops but only %d distinct asset(s); the full range contains %d distinct asset(s), and at least one first appears in skipped ledger %d",
		startLedger,
		endLedger,
		limit,
		len(limitedOps),
		len(limitedAssets),
		len(allAssets),
		endLedger,
	)
}

func distinctAssets(t *testing.T, ops []input.AssetTransformInput) (map[int64]struct{}, map[int64]uint32) {
	t.Helper()

	distinct := make(map[int64]struct{})
	firstSeenLedger := make(map[int64]uint32)
	for _, op := range ops {
		asset, err := transform.TransformAsset(
			op.Operation,
			op.OperationIndex,
			op.TransactionIndex,
			op.LedgerSeqNum,
			op.LedgerCloseMeta,
		)
		if err != nil {
			t.Fatalf("TransformAsset() error = %v", err)
		}
		distinct[asset.AssetID] = struct{}{}
		if _, ok := firstSeenLedger[asset.AssetID]; !ok {
			firstSeenLedger[asset.AssetID] = uint32(op.LedgerSeqNum)
		}
	}

	return distinct, firstSeenLedger
}
```

## Expected vs Actual Behavior

- **Expected**: `--limit N` should keep reading until it can emit up to `N` distinct asset rows or exhaust the range.
- **Actual**: `GetPaymentOperations()` stops after `N` qualifying operations at ledger granularity, and `export_assets` then deduplicates that slice down to fewer than `N` rows.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the production datastore-backed asset reader and the production asset transform on immutable pubnet ledgers.
2. Realistic preconditions: YES — the reproducer uses normal mainnet history where ledger `30820391` contains repeated native-asset references and ledger `30820392` introduces additional assets.
3. Bug vs by-design: BUG — the help text promises a maximum number of exported assets, not a maximum number of qualifying operations before deduplication.
4. Final severity: High — the export silently omits distinct asset rows that are present in-range, producing structurally wrong asset datasets.
5. In scope: YES — this is a concrete wrong-output path in the production export pipeline.
6. Test correctness: CORRECT — the fixed PoC does not hand-simulate truncation; it exercises the real reader, the real transform, and proves the later distinct assets exist in the same requested range.
7. Alternative explanations: NONE — once the reader spends the limit on duplicate-heavy input and the exporter deduplicates afterward, underfilled output follows directly.
8. Novelty: NOVEL — although related to the previously confirmed within-ledger overshoot bug, this is a distinct export-layer omission mode caused by post-read deduplication.

## Suggested Fix

Apply the limit to distinct emitted asset rows instead of raw `AssetTransformInput` items. The safest fix is to move distinct-asset counting to the exporter (or teach the reader to track seen asset IDs while scanning) so duplicate-heavy ledgers cannot consume the budget before later new assets are read.
