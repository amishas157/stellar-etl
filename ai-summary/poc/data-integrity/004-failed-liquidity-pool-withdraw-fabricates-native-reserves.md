# H004: Failed liquidity-pool withdrawals fabricate `native/native` reserve assets

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a failed `liquidity_pool_withdraw` row exports reserve-asset metadata, `reserve_a_*` and `reserve_b_*` should still identify the actual pool assets, or else those fields should remain unset. A failed withdrawal against a credit/native pool should never serialize both reserve assets as native.

## Mechanism

The withdraw branch also initializes `assetA` and `assetB` as zero-value `xdr.Asset` structs and only populates them inside the success-only block. On failure it still calls `addAssetDetailsToOperationDetails()` for both reserve slots, and the zero enum value again maps to `AssetTypeNative`, so the ETL emits a believable but false `native/native` pair.

## Trigger

1. Export a failed `liquidity_pool_withdraw` against any pool whose reserves are not both native.
2. Inspect `history_operations.details.reserve_a_asset_type`, `reserve_b_asset_type`, and the reserve asset IDs.
3. Observe that both reserve assets are exported as native even though the pool identified by `liquidity_pool_id` uses a different pair.

## Target Code

- `internal/transform/operation.go:extractOperationDetails:1022-1061` — failed withdraw path leaves `assetA` / `assetB` at zero values
- `internal/transform/operation.go:addAssetDetailsToOperationDetails:367-386` — writes native asset metadata for zero-value assets
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/asset.go:Extract:298-342` — zero-value asset extraction behavior
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:1869` — `AssetTypeNative` is enum value `0`

## Evidence

The only place the withdraw branch learns the pool assets is the success-only `getLiquidityPoolAndProductDelta()` block. When that block is skipped, both reserve asset structs stay zero-valued but are still serialized through the generic asset helper, which deterministically reports them as native assets.

## Anti-Evidence

As with failed deposits, the pool identifier fields remain correct and could let a downstream consumer reconstruct the real asset pair from another export. The immediate withdrawal row still contains wrong reserve-asset metadata, which is enough for silent structural corruption in any consumer that trusts the row at face value.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full execution path through the `LiquidityPoolWithdraw` branch (lines 1021-1061) in `extractOperationDetails()`. The code declares `assetA, assetB xdr.Asset` as zero-value structs (line 1026), which default to `AssetTypeAssetTypeNative` (enum value 0). These are only populated inside the `if transaction.Result.Successful()` block (line 1037-1046). On failure, the zero-value assets are passed to `addAssetDetailsToOperationDetails()` (lines 1048, 1055), which calls `asset.Extract()` and serializes them as `asset_type: "native"` with the hardcoded native `asset_id` of `-5706705804583548011`. The identical pattern applies to the `LiquidityPoolDeposit` branch (lines 957-1019). Prior investigations (024, 025 in `fail/data-input/`) examined zero **amounts** for failed LP operations and concluded working-as-designed, but never addressed the fabricated **asset types** — which is a materially distinct issue.

### Code Paths Examined

- `internal/transform/operation.go:1021-1028` — `LiquidityPoolWithdraw` declares `assetA, assetB xdr.Asset` as zero-value (Go default: `AssetTypeAssetTypeNative`, enum 0)
- `internal/transform/operation.go:1037-1046` — Success-only block populates `assetA`, `assetB` from pool params. On failure this entire block is skipped.
- `internal/transform/operation.go:1048-1056` — `addAssetDetailsToOperationDetails(details, assetA, "reserve_a")` and `addAssetDetailsToOperationDetails(details, assetB, "reserve_b")` called unconditionally with potentially zero-value assets.
- `internal/transform/operation.go:367-386` — `addAssetDetailsToOperationDetails` calls `asset.Extract()`, gets `assetType="native"` for zero-value asset. Sets `result["reserve_a_asset_type"] = "native"` and `result["reserve_a_asset_id"] = int64(-5706705804583548011)`.
- `internal/transform/operation.go:957-984` — `LiquidityPoolDeposit` has the identical pattern: `assetA, assetB` zero-value, populated only on success, serialized as native on failure.
- `xdr/xdr_generated.go:1869` — Confirms `AssetTypeAssetTypeNative = 0` (Go zero-value for the enum)

### Findings

The bug is confirmed: for any failed `liquidity_pool_withdraw` (or `liquidity_pool_deposit`) against a pool that contains non-native assets, the exported `reserve_a_asset_type`, `reserve_a_asset_code`, `reserve_a_asset_issuer`, `reserve_a_asset_id` (and the `reserve_b_*` counterparts) are fabricated as native. This is structural corruption:

1. **The pool's constituent assets are immutable** — they don't depend on whether the operation succeeded. A USDC/XLM pool is always USDC/XLM.
2. **The code comment at line 1038 says "omitted asset"** — suggesting the developers intended the asset fields to be absent, but the implementation serializes the Go zero-value as native instead of truly omitting.
3. **This affects the vast majority of LP operations** — most Stellar liquidity pools pair a non-native credit asset with XLM. Any failed operation against such a pool produces wrong asset metadata.
4. **The `asset_id` is also wrong** — both reserves get the hardcoded native `asset_id` of `-5706705804583548011` instead of the `FarmHashAsset()` result for the actual credit asset.
5. **Novelty vs prior investigations** — The 024/025 fail entries examined zero amounts and concluded working-as-designed. Those verdicts are correct for amounts (zero accurately reflects no transfer). But asset types are a separate concern: the pool's assets are fixed properties, and emitting `native/native` for a USDC/XLM pool is factually wrong regardless of operation success.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go`
- **Setup**: Create a mock failed `LiquidityPoolWithdraw` transaction (set `transaction.Result.Successful()` to return false). Use a pool with one credit asset (e.g., USDC:GISSUER) and native XLM.
- **Steps**: Call `TransformOperation()` with the failed transaction and verify the `details` map.
- **Assertion**: Assert that `details["reserve_a_asset_type"]` equals `"native"` AND `details["reserve_b_asset_type"]` equals `"native"` — demonstrating that both reserve assets are incorrectly serialized as native even though the pool has a non-native asset. Optionally, also verify the same behavior in `LiquidityPoolDeposit` to show this is a cross-operation pattern.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestFailedLPWithdrawFabricatesNativeReserves"
**Test Language**: Go

### Demonstration

The test constructs a failed `LiquidityPoolWithdraw` transaction with an arbitrary pool ID and calls `extractOperationDetails()`. Since the transaction fails, the success-only block that populates `assetA`/`assetB` from pool metadata is skipped. The zero-value `xdr.Asset` structs (which default to `AssetTypeAssetTypeNative`, enum 0) are then serialized by `addAssetDetailsToOperationDetails()`, causing both `reserve_a_asset_type` and `reserve_b_asset_type` to be reported as `"native"` with the hardcoded native sentinel `asset_id` of `-5706705804583548011`. This proves the bug: any failed LP withdraw against a non-native pool fabricates wrong asset metadata.

### Test Body

```go
func TestFailedLPWithdrawFabricatesNativeReserves(t *testing.T) {
	poolID := xdr.PoolId{1, 2, 3, 4, 5, 6, 7, 8, 9}

	lpWithdrawOp := xdr.Operation{
		SourceAccount: nil,
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

	failedResultMeta := utils.CreateSampleResultMeta(false, 1)

	envelope := xdr.TransactionV1Envelope{
		Tx: xdr.Transaction{
			SourceAccount: testAccount3,
			Operations:    []xdr.Operation{lpWithdrawOp},
		},
	}

	tx := ingest.LedgerTransaction{
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

	if tx.Result.Successful() {
		t.Fatal("expected transaction to be failed")
	}

	details, err := extractOperationDetails(lpWithdrawOp, tx, 0, "Test SDF Network ; September 2015")
	if err != nil {
		t.Fatalf("extractOperationDetails returned error: %v", err)
	}

	reserveAType, _ := details["reserve_a_asset_type"].(string)
	reserveBType, _ := details["reserve_b_asset_type"].(string)

	if reserveAType != "native" || reserveBType != "native" {
		t.Fatalf("expected fabricated native/native but got reserve_a=%q reserve_b=%q",
			reserveAType, reserveBType)
	}

	reserveAID, _ := details["reserve_a_asset_id"].(int64)
	reserveBID, _ := details["reserve_b_asset_id"].(int64)

	nativeSentinel := int64(-5706705804583548011)
	if reserveAID != nativeSentinel || reserveBID != nativeSentinel {
		t.Fatalf("expected native sentinel asset_id=%d for both reserves, got a=%d b=%d",
			nativeSentinel, reserveAID, reserveBID)
	}

	t.Logf("BUG CONFIRMED: failed LP withdraw fabricates native/native reserves")
	t.Logf("  reserve_a_asset_type=%q  reserve_a_asset_id=%d", reserveAType, reserveAID)
	t.Logf("  reserve_b_asset_type=%q  reserve_b_asset_id=%d", reserveBType, reserveBID)
}
```

### Test Output

```
=== RUN   TestFailedLPWithdrawFabricatesNativeReserves
    data_integrity_poc_test.go:161: BUG CONFIRMED: failed LP withdraw fabricates native/native reserves
    data_integrity_poc_test.go:162:   reserve_a_asset_type="native"  reserve_a_asset_id=-5706705804583548011
    data_integrity_poc_test.go:163:   reserve_b_asset_type="native"  reserve_b_asset_id=-5706705804583548011
--- PASS: TestFailedLPWithdrawFabricatesNativeReserves (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.478s
```
