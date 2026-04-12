# 026: Pool-share trustlines collapse distinct pools into one `asset_id`

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformTrustline()` exports the same `asset_id` for every pool-share trustline, even when the rows reference different liquidity pools. The transform correctly emits distinct `liquidity_pool_id` values, but the surrogate `asset_id` drops that identity component and silently collapses all pool-share trustlines into one bucket.

## Root Cause

For `AssetTypeAssetTypePoolShare`, `TransformTrustline()` sets `asset_type` and `liquidity_pool_id` but leaves `asset_code` and `asset_issuer` empty. It then calls `FarmHashAsset(outputAssetCode, outputAssetIssuer, asset.Type.String())`, so every pool-share trustline hashes the same `("", "", "AssetTypePoolShare")` tuple even though the underlying XDR asset is distinguished by `LiquidityPoolId`.

## Reproduction

1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
3. Run `go test ./internal/transform/... -run TestPoolShareTrustlinesCollapseAssetID -v`.
4. Observe that two trustlines with different `liquidity_pool_id` values both export `asset_id = -1967220342708457407`.

## Affected Code

- `internal/transform/trustline.go:TransformTrustline:43-57` — pool-share trustlines populate `liquidity_pool_id` but hash `asset_id` from empty code/issuer plus the raw enum type.
- `internal/transform/asset.go:FarmHashAsset:72-76` — the hash helper only incorporates code, issuer, and type, so omitted pool IDs can never affect the result.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestPoolShareTrustlinesCollapseAssetID`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestPoolShareTrustlinesCollapseAssetID demonstrates that two pool-share
// trustlines referencing different liquidity pools produce the same asset_id,
// because FarmHashAsset is called with empty code/issuer and the pool ID is
// never included in the hash input.
func TestPoolShareTrustlinesCollapseAssetID(t *testing.T) {
	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}

	poolA := xdr.PoolId{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32}
	poolB := xdr.PoolId{99, 98, 97, 96, 95, 94, 93, 92, 91, 90, 89, 88, 87, 86, 85, 84, 83, 82, 81, 80, 79, 78, 77, 76, 75, 74, 73, 72, 71, 70, 69, 68}

	makeLedgerEntry := func(account xdr.AccountId, poolID xdr.PoolId) xdr.LedgerEntry {
		return xdr.LedgerEntry{
			LastModifiedLedgerSeq: 100,
			Data: xdr.LedgerEntryData{
				Type: xdr.LedgerEntryTypeTrustline,
				TrustLine: &xdr.TrustLineEntry{
					AccountId: account,
					Asset: xdr.TrustLineAsset{
						Type:            xdr.AssetTypeAssetTypePoolShare,
						LiquidityPoolId: &poolID,
					},
					Balance: 1000000,
					Limit:   9000000000000000000,
					Flags:   xdr.Uint32(xdr.TrustLineFlagsAuthorizedFlag),
					Ext: xdr.TrustLineEntryExt{
						V: 1,
						V1: &xdr.TrustLineEntryV1{
							Liabilities: xdr.Liabilities{},
						},
					},
				},
			},
		}
	}

	entryA := makeLedgerEntry(testAccount1ID, poolA)
	entryB := makeLedgerEntry(testAccount2ID, poolB)

	changeA := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeTrustline,
		Post:       &entryA,
	}
	changeB := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeTrustline,
		Post:       &entryB,
	}

	outputA, err := TransformTrustline(changeA, header)
	if err != nil {
		t.Fatalf("TransformTrustline(A) error: %v", err)
	}
	outputB, err := TransformTrustline(changeB, header)
	if err != nil {
		t.Fatalf("TransformTrustline(B) error: %v", err)
	}

	if outputA.LiquidityPoolID == outputB.LiquidityPoolID {
		t.Fatalf("test setup error: both entries have the same LiquidityPoolID %s", outputA.LiquidityPoolID)
	}

	t.Logf("Pool A LiquidityPoolID: %s", outputA.LiquidityPoolID)
	t.Logf("Pool B LiquidityPoolID: %s", outputB.LiquidityPoolID)
	t.Logf("Pool A AssetID: %d", outputA.AssetID)
	t.Logf("Pool B AssetID: %d", outputB.AssetID)

	if outputA.AssetID == outputB.AssetID {
		t.Errorf("BUG CONFIRMED: Two pool-share trustlines with different LiquidityPoolIDs produce the same AssetID (%d). FarmHashAsset ignores the pool ID, collapsing all pool shares into one asset_id bucket.", outputA.AssetID)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Distinct pool-share trustlines should export distinct `asset_id` values because the underlying asset identity includes `liquidity_pool_id`.
- **Actual**: Distinct pool-share trustlines export the same `asset_id` whenever their type is `pool_share`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls `TransformTrustline()` directly on two real pool-share `ingest.Change` values with different pool IDs.
2. Realistic preconditions: YES — any ledger containing trustlines for more than one liquidity pool can produce this output; the test uses normal production types and code paths.
3. Bug vs by-design: BUG — upstream XDR equality keys pool-share trustlines by `LiquidityPoolId`, and sibling ETL paths surface `liquidity_pool_id` for liquidity-pool-share assets instead of collapsing them.
4. Final severity: High — the exported surrogate key is structurally wrong, causing silent conflation of distinct non-financial entities.
5. In scope: YES — this is a production transform-layer export error, not a theoretical or test-only issue.
6. Test correctness: CORRECT — the test first proves the inputs are different, then fails only when the production output aliases them.
7. Alternative explanations: NONE — `TransformTrustline()` hashes identical inputs for every pool-share trustline, which fully explains the reproduced collision.
8. Novelty: UNASSESSED — duplicate handling is performed by the orchestrator.

## Suggested Fix

Include the pool ID in the trustline `asset_id` derivation for `AssetTypeAssetTypePoolShare`, either by hashing `liquidity_pool_id` directly or by using a pool-share-specific asset-key helper that mirrors the XDR identity semantics.
