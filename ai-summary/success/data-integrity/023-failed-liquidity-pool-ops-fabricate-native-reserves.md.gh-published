# 023: Failed Liquidity-Pool Ops Fabricate Native Reserves

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

Failed `liquidity_pool_deposit` and `liquidity_pool_withdraw` rows export `reserve_a_*` and `reserve_b_*` as if both reserves were the native asset, even when the referenced pool ID belongs to a non-native pair. The ETL preserves the correct `liquidity_pool_id`, but the companion reserve-asset columns are silently corrupted into a plausible-looking `native/native` pair with the native sentinel asset ID.

This is a real transform bug, not a test artifact. A strengthened PoC that constructs a canonical pool ID from two actual non-native assets still produces `reserve_a_asset_type="native"` and `reserve_b_asset_type="native"` on the failed branch because the code serializes zero-value `xdr.Asset{}` values as real native assets.

## Root Cause

In the failed branches of `extractOperationDetails()`, the liquidity-pool deposit and withdraw cases leave `assetA` and `assetB` at their Go zero values because metadata lookup only runs when `transaction.Result.Successful()` is true. Those zero-value `xdr.Asset{}` structs have `Type == 0`, which upstream XDR defines as `AssetTypeNative`.

`addAssetDetailsToOperationDetails()` is then called unconditionally for both reserve slots. It runs `asset.Extract()`, writes `asset_type="native"`, and assigns the real native `asset_id` sentinel `-5706705804583548011`, so the exporter fabricates concrete reserve-asset metadata instead of omitting unknown assets.

## Reproduction

Any normal export that includes a failed liquidity-pool deposit or withdraw against a non-native pool can hit this path. Because failed operations do not populate pool metadata, the ETL cannot recover the real reserve assets from the operation body alone, yet it still emits concrete reserve-asset fields; downstream joins or filters that trust those columns will see a false `native/native` pool.

## Affected Code

- `internal/transform/operation.go:addAssetDetailsToOperationDetails:367-386` — zero-value `xdr.Asset` is extracted and serialized as a real native asset with the native sentinel `asset_id`
- `internal/transform/operation.go:extractOperationDetails:957-1059` — failed liquidity-pool deposit/withdraw branches leave `assetA` and `assetB` unset but still serialize both reserve slots unconditionally

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestFailedLPDepositFabricatesNativeReserves`, `TestFailedLPWithdrawFabricatesNativeReserves`
- **Test language**: `go`
- **How to run**: `cd <repo-root> && go build ./... && go test ./internal/transform/... -run 'TestFailedLP(Deposit|Withdraw)FabricatesNativeReserves' -v` after creating the target test file with the body below.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
	"github.com/stellar/stellar-etl/v2/internal/utils"
)

func failedTransactionWithOperation(op xdr.Operation) ingest.LedgerTransaction {
	failedResultMeta := utils.CreateSampleResultMeta(false, 1)
	envelope := xdr.TransactionV1Envelope{
		Tx: xdr.Transaction{
			SourceAccount: testAccount3,
			Operations:    []xdr.Operation{op},
		},
	}

	return ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1:   &envelope,
		},
		Result: failedResultMeta.Result,
		UnsafeMeta: xdr.TransactionMeta{
			V:  1,
			V1: &xdr.TransactionMetaV1{},
		},
	}
}

func nonNativePoolID(t *testing.T) (xdr.PoolId, xdr.Asset, xdr.Asset) {
	t.Helper()

	assetA, assetB := usdtAsset, ethAsset
	if assetB.LessThan(assetA) {
		assetA, assetB = assetB, assetA
	}

	poolID, err := xdr.NewPoolId(assetA, assetB, 30)
	if err != nil {
		t.Fatalf("failed to create realistic non-native pool id: %v", err)
	}

	return poolID, assetA, assetB
}

func assertFabricatedNativeReserves(t *testing.T, details map[string]interface{}, actualA, actualB xdr.Asset) {
	t.Helper()

	nativeSentinel := int64(-5706705804583548011)
	for prefix, asset := range map[string]xdr.Asset{
		"reserve_a": actualA,
		"reserve_b": actualB,
	} {
		var actualType, actualCode, actualIssuer string
		if err := asset.Extract(&actualType, &actualCode, &actualIssuer); err != nil {
			t.Fatalf("failed to extract %s asset details: %v", prefix, err)
		}
		if actualType == "native" {
			t.Fatalf("test setup is invalid: %s unexpectedly resolved to native", prefix)
		}

		gotType, _ := details[prefix+"_asset_type"].(string)
		if gotType != "native" {
			t.Fatalf("expected fabricated native %s for pool asset %s:%s, got %q",
				prefix, actualCode, actualIssuer, gotType)
		}

		gotID, _ := details[prefix+"_asset_id"].(int64)
		if gotID != nativeSentinel {
			t.Fatalf("expected native sentinel asset_id=%d for %s, got %d",
				nativeSentinel, prefix, gotID)
		}

		if _, ok := details[prefix+"_asset_code"]; ok {
			t.Fatalf("expected no %s_asset_code for fabricated native reserve", prefix)
		}
		if _, ok := details[prefix+"_asset_issuer"]; ok {
			t.Fatalf("expected no %s_asset_issuer for fabricated native reserve", prefix)
		}
	}
}

func TestFailedLPDepositFabricatesNativeReserves(t *testing.T) {
	poolID, actualA, actualB := nonNativePoolID(t)
	lpDepositOp := xdr.Operation{
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeLiquidityPoolDeposit,
			LiquidityPoolDepositOp: &xdr.LiquidityPoolDepositOp{
				LiquidityPoolId: poolID,
				MaxAmountA:      1000,
				MaxAmountB:      100,
				MinPrice:        xdr.Price{N: 1, D: 1000000},
				MaxPrice:        xdr.Price{N: 1000000, D: 1},
			},
		},
	}

	tx := failedTransactionWithOperation(lpDepositOp)
	if tx.Result.Successful() {
		t.Fatal("expected transaction to be failed")
	}

	details, err := extractOperationDetails(lpDepositOp, tx, 0, "Test SDF Network ; September 2015")
	if err != nil {
		t.Fatalf("extractOperationDetails returned error: %v", err)
	}

	if got, _ := details["liquidity_pool_id"].(string); got != PoolIDToString(poolID) {
		t.Fatalf("expected operation details to retain pool id %q, got %q", PoolIDToString(poolID), got)
	}

	assertFabricatedNativeReserves(t, details, actualA, actualB)
}

func TestFailedLPWithdrawFabricatesNativeReserves(t *testing.T) {
	poolID, actualA, actualB := nonNativePoolID(t)
	lpWithdrawOp := xdr.Operation{
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeLiquidityPoolWithdraw,
			LiquidityPoolWithdrawOp: &xdr.LiquidityPoolWithdrawOp{
				LiquidityPoolId: poolID,
				Amount:          4,
				MinAmountA:      1,
				MinAmountB:      1,
			},
		},
	}

	tx := failedTransactionWithOperation(lpWithdrawOp)
	if tx.Result.Successful() {
		t.Fatal("expected transaction to be failed")
	}

	details, err := extractOperationDetails(lpWithdrawOp, tx, 0, "Test SDF Network ; September 2015")
	if err != nil {
		t.Fatalf("extractOperationDetails returned error: %v", err)
	}

	if got, _ := details["liquidity_pool_id"].(string); got != PoolIDToString(poolID) {
		t.Fatalf("expected operation details to retain pool id %q, got %q", PoolIDToString(poolID), got)
	}

	assertFabricatedNativeReserves(t, details, actualA, actualB)
}
```

## Expected vs Actual Behavior

- **Expected**: Failed liquidity-pool deposit/withdraw rows should either omit reserve-asset details entirely or reflect the actual pool assets; they should never claim a concrete native asset unless the pool really contains native reserves.
- **Actual**: Failed rows serialize both reserve slots as `asset_type="native"` with the native sentinel `asset_id`, even when the pool ID belongs to a non-native asset pair.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the production `extractOperationDetails()` path for failed liquidity-pool deposit and withdraw operations and observes the wrong exported reserve-asset fields directly.
2. Realistic preconditions: YES — failed LP operations occur in normal Stellar usage, and the strengthened test uses a canonical pool ID generated by `xdr.NewPoolId()` from two real non-native assets rather than an invented byte array.
3. Bug vs by-design: BUG — the code comment says failed LP operations should use "omitted asset and 0 amounts," and there is no documented "unknown means native" sentinel; emitting a real native asset is misleading, not intentional.
4. Final severity: High — the corrupted fields are structural/identity data rather than a wrong numeric amount, but they are persisted export columns that can poison downstream asset joins and analytics.
5. In scope: YES — this is a concrete ETL export corruption in repository code, not an SDK bug or a purely theoretical concern.
6. Test correctness: CORRECT — the final PoC uses production helpers, production XDR types, and a realistic pool identifier; it passes because the transform really serializes zero-value assets as native, not because the assertions restate the setup.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Do not serialize reserve-asset details when the transaction failed and the pool metadata was never resolved. The safest fix is to gate the `addAssetDetailsToOperationDetails()` calls behind a success/metadata-present check or switch `assetA`/`assetB` to an explicit optional representation so the failed branch can omit those fields instead of passing zero-value `xdr.Asset{}` into the native-asset serializer.
