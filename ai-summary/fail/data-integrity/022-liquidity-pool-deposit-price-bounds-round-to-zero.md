# H002: Tiny liquidity-pool deposit price bounds collapse to zero

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_operations.details.min_price` and `history_operations.details.max_price` for `liquidity_pool_deposit` should preserve the actual nonzero slippage bounds encoded in the operation. If a transaction sets `MinPrice = 1/2147483647` or `MaxPrice = 1/2147483647`, the exported numeric bound should remain nonzero.

## Mechanism

The `liquidity_pool_deposit` branch delegates both bound fields to `addPriceDetails()`, which round-trips `xdr.Price` through `Price.String()` and `strconv.ParseFloat`. Because `Price.String()` is only 7-decimal precision, very small but valid bounds become `"0.0000000"` and the ETL exports `min_price = 0` and/or `max_price = 0`, silently widening the apparent slippage range.

## Trigger

1. Export a successful `liquidity_pool_deposit` whose `MinPrice` or `MaxPrice` is a valid tiny rational such as `1/2147483647`.
2. Inspect `history_operations.details.min_price` / `max_price`.
3. Observe that the numeric field is `0` even though `min_price_r` / `max_price_r` retain the exact rational components.

## Target Code

- `internal/transform/operation.go:addPriceDetails:409-419` — lossy `Price.String()` -> `ParseFloat` conversion
- `internal/transform/operation.go:extractOperationDetails:957-1013` — `liquidity_pool_deposit` writes `min_price` and `max_price` via that helper
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/price.go:String:7-10` — formatter truncates to 7 decimal places

## Evidence

The deposit branch uses `addPriceDetails(details, op.MinPrice, "min")` and `addPriceDetails(details, op.MaxPrice, "max")`, so it inherits the same 7-decimal string truncation as offer operations. Any legitimate bound smaller than `5e-8` is rendered as `"0.0000000"` before parsing, making the exported numeric guardrail look disabled.

## Anti-Evidence

The exact rationals survive separately in `min_price_r` and `max_price_r`, so the raw envelope data is not lost completely. But the primary numeric bound fields that downstream analytics are most likely to read still become wrong.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete code path from `extractOperationDetails` LP deposit branch (operation.go:957-1013) through `addPriceDetails` (operation.go:409-421) into `xdr.Price.String()` (go-stellar-sdk xdr/price.go:8-10). Confirmed that `Price.String()` delegates to `big.NewRat(N, D).FloatString(7)`, which produces exactly 7 decimal places. Verified with actual Go execution that `Price{1, 20000001}` produces `"0.0000000"` → `ParseFloat` returns `0.0`. The same `addPriceDetails` function is also called for ManageBuyOffer (line 709), ManageSellOffer (line 728), and CreatePassiveSellOffer (line 746) prices, making this a cross-operation issue.

### Code Paths Examined

- `internal/transform/operation.go:409-421` — `addPriceDetails()` calls `price.String()` then `strconv.ParseFloat(s, 64)`, assigning result to `prefix+"price"`. Also sets `prefix+"price_r"` with exact N/D.
- `internal/transform/operation.go:1008-1012` — LP deposit calls `addPriceDetails(details, op.MinPrice, "min")` and `addPriceDetails(details, op.MaxPrice, "max")`
- `go-stellar-sdk xdr/price.go:8-10` — `Price.String()` returns `big.NewRat(int64(p.N), int64(p.D)).FloatString(7)` — exactly 7 decimal places
- `internal/transform/operation.go:709,728,746` — same `addPriceDetails` used for ManageBuyOffer, ManageSellOffer, CreatePassiveSellOffer prices

### Findings

**Confirmed**: Any `xdr.Price` where the rational value N/D < 5×10⁻⁸ (approximately D > 20,000,000×N) will be exported as `0.0` in the scalar price field while the companion `*_price_r` rational preserves the exact value.

**Go verification**: Tested with `big.NewRat(1, 20000001).FloatString(7)` → `"0.0000000"` → `ParseFloat` → `0.0`. Boundary confirmed: `Price{1, 20000000}` → `"0.0000001"` (survives), `Price{1, 20000001}` → `"0.0000000"` (truncated to zero).

**Scope**: Affects 5 call sites — LP deposit min_price, LP deposit max_price, ManageBuyOffer price, ManageSellOffer price, CreatePassiveSellOffer price. All share the same `addPriceDetails` code path.

**Severity rationale**: Downgraded from Critical to Medium because: (1) affected prices are extremely small (< 5×10⁻⁸), limiting real-world prevalence; (2) exact rational components survive in companion `*_price_r` fields; (3) for LP deposit slippage bounds, a price this small is functionally near-zero. However, the bug is real — a nonzero financial field becomes exactly zero, which is semantically incorrect and could mislead downstream analytics that read only the scalar price field.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go`
- **Setup**: Create a `LiquidityPoolDepositOp` with `MinPrice: xdr.Price{N: 1, D: 20000001}` and `MaxPrice: xdr.Price{N: 1, D: 2147483647}`. Wrap in a successful transaction with appropriate LP metadata.
- **Steps**: Call `TransformOperation()` on the constructed operation. Extract `details["min_price"]` and `details["max_price"]` from the result.
- **Assertion**: Assert that `details["min_price"]` equals `0.0` (demonstrating the bug). Also assert that `details["min_price_r"]` correctly preserves `{N: 1, D: 20000001}` (showing the data loss is in the scalar field only). Optionally, also test with `ManageSellOffer` using `Price{1, 20000001}` to show the cross-operation scope.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestLPDepositTinyPriceBoundsCollapseToZero"
**Test Language**: Go

### Demonstration

The test constructs a `LiquidityPoolDepositOp` with tiny price bounds (`MinPrice: {1, 20000001}` and `MaxPrice: {1, 2147483647}`) and passes it through `TransformOperation()`. The scalar `min_price` and `max_price` fields are both exported as `0.0` because `addPriceDetails()` calls `Price.String()` which uses `FloatString(7)`, truncating values below 5×10⁻⁸ to `"0.0000000"`. Meanwhile, the companion rational fields (`min_price_r`, `max_price_r`) correctly preserve the exact numerator/denominator, proving the data loss is in the scalar conversion path only.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestLPDepositTinyPriceBoundsCollapseToZero demonstrates that
// addPriceDetails truncates very small xdr.Price values to 0.0
// because Price.String() uses FloatString(7) — only 7 decimal places.
// Any rational N/D < 5e-8 is rendered as "0.0000000" → ParseFloat → 0.0.
func TestLPDepositTinyPriceBoundsCollapseToZero(t *testing.T) {
	sourceAccount, _ := xdr.NewMuxedAccount(
		xdr.CryptoKeyTypeKeyTypeEd25519, xdr.Uint256([32]byte{}),
	)

	tinyMinPrice := xdr.Price{N: 1, D: 20000001} // ~5e-8, below 7-decimal threshold
	tinyMaxPrice := xdr.Price{N: 1, D: 2147483647} // ~4.66e-10, far below threshold

	lpDepositOp := xdr.Operation{
		SourceAccount: &sourceAccount,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeLiquidityPoolDeposit,
			LiquidityPoolDepositOp: &xdr.LiquidityPoolDepositOp{
				LiquidityPoolId: xdr.PoolId{1, 2, 3},
				MaxAmountA:      1000,
				MaxAmountB:      1000,
				MinPrice:        tinyMinPrice,
				MaxPrice:        tinyMaxPrice,
			},
		},
	}

	// Use a failed transaction so we don't need LP metadata for the delta lookup.
	// The price extraction in addPriceDetails happens unconditionally.
	failedResult := xdr.TransactionResultPair{
		Result: xdr.TransactionResult{
			Result: xdr.TransactionResultResult{
				Code: xdr.TransactionResultCodeTxFailed,
				Results: &[]xdr.OperationResult{
					{
						Code: xdr.OperationResultCodeOpInner,
						Tr: &xdr.OperationResultTr{
							Type: xdr.OperationTypeLiquidityPoolDeposit,
							LiquidityPoolDepositResult: &xdr.LiquidityPoolDepositResult{
								Code: xdr.LiquidityPoolDepositResultCodeLiquidityPoolDepositUnderfunded,
							},
						},
					},
				},
			},
		},
	}

	envelope := xdr.TransactionV1Envelope{
		Tx: xdr.Transaction{
			SourceAccount: sourceAccount,
			Operations:    []xdr.Operation{lpDepositOp},
			Ext: xdr.TransactionExt{
				V: 0,
			},
		},
	}

	tx := ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1:   &envelope,
		},
		Result: failedResult,
		UnsafeMeta: xdr.TransactionMeta{
			V: 1,
			V1: &xdr.TransactionMetaV1{
				Operations: []xdr.OperationMeta{
					{Changes: xdr.LedgerEntryChanges{}},
				},
			},
		},
	}

	ledgerCloseMeta := xdr.LedgerCloseMeta{
		V: 0,
		V0: &xdr.LedgerCloseMetaV0{
			LedgerHeader: xdr.LedgerHeaderHistoryEntry{
				Header: xdr.LedgerHeader{
					ScpValue: xdr.StellarValue{CloseTime: 0},
					LedgerSeq: 0,
				},
			},
		},
	}

	output, err := TransformOperation(lpDepositOp, 0, tx, 0, ledgerCloseMeta, "")
	if err != nil {
		t.Fatalf("TransformOperation returned unexpected error: %v", err)
	}

	details := output.OperationDetails

	// --- Demonstrate the bug: scalar min_price is truncated to 0.0 ---
	minPrice, ok := details["min_price"].(float64)
	if !ok {
		t.Fatalf("min_price not found or not float64: %v", details["min_price"])
	}
	if minPrice != 0.0 {
		t.Errorf("Expected min_price to be truncated to 0.0, got %v", minPrice)
	}
	t.Logf("BUG CONFIRMED: min_price = %v (should be ~%e)", minPrice, float64(1)/float64(20000001))

	maxPrice, ok := details["max_price"].(float64)
	if !ok {
		t.Fatalf("max_price not found or not float64: %v", details["max_price"])
	}
	if maxPrice != 0.0 {
		t.Errorf("Expected max_price to be truncated to 0.0, got %v", maxPrice)
	}
	t.Logf("BUG CONFIRMED: max_price = %v (should be ~%e)", maxPrice, float64(1)/float64(2147483647))

	// --- Verify that the rational forms preserve the exact values ---
	minPriceR, ok := details["min_price_r"].(Price)
	if !ok {
		t.Fatalf("min_price_r not found or not Price: %v", details["min_price_r"])
	}
	if minPriceR.Numerator != 1 || minPriceR.Denominator != 20000001 {
		t.Errorf("min_price_r corrupted: got {%d, %d}, want {1, 20000001}",
			minPriceR.Numerator, minPriceR.Denominator)
	}
	t.Logf("min_price_r preserved: {N: %d, D: %d}", minPriceR.Numerator, minPriceR.Denominator)

	maxPriceR, ok := details["max_price_r"].(Price)
	if !ok {
		t.Fatalf("max_price_r not found or not Price: %v", details["max_price_r"])
	}
	if maxPriceR.Numerator != 1 || maxPriceR.Denominator != 2147483647 {
		t.Errorf("max_price_r corrupted: got {%d, %d}, want {1, 2147483647}",
			maxPriceR.Numerator, maxPriceR.Denominator)
	}
	t.Logf("max_price_r preserved: {N: %d, D: %d}", maxPriceR.Numerator, maxPriceR.Denominator)

	// --- Summary: nonzero prices become zero in scalar fields, but rationals survive ---
	t.Logf("SUMMARY: Scalar price fields lose precision via Price.String() → FloatString(7) → ParseFloat")
	t.Logf("  min_price: 1/20000001 ≈ 5.0e-08 → exported as %v", minPrice)
	t.Logf("  max_price: 1/2147483647 ≈ 4.66e-10 → exported as %v", maxPrice)
}
```

### Test Output

```
=== RUN   TestLPDepositTinyPriceBoundsCollapseToZero
    data_integrity_poc_test.go:111: BUG CONFIRMED: min_price = 0 (should be ~5.000000e-08)
    data_integrity_poc_test.go:120: BUG CONFIRMED: max_price = 0 (should be ~4.656613e-10)
    data_integrity_poc_test.go:131: min_price_r preserved: {N: 1, D: 20000001}
    data_integrity_poc_test.go:141: max_price_r preserved: {N: 1, D: 2147483647}
    data_integrity_poc_test.go:144: SUMMARY: Scalar price fields lose precision via Price.String() → FloatString(7) → ParseFloat
    data_integrity_poc_test.go:145:   min_price: 1/20000001 ≈ 5.0e-08 → exported as 0
    data_integrity_poc_test.go:146:   max_price: 1/2147483647 ≈ 4.66e-10 → exported as 0
--- PASS: TestLPDepositTinyPriceBoundsCollapseToZero (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.895s
```

---

## Final Review

**Verdict**: REJECTED
**Date**: 2026-04-12
**Final review by**: gpt-5.4, high
**Failed At**: final-review

### Adversarial Analysis

1. **Exercises claimed issue**: PASS — the PoC drives `TransformOperation()` through `extractOperationDetails()` into `addPriceDetails()` and correctly shows that `strconv.ParseFloat(price.String(), 64)` turns very small valid `xdr.Price` values into `0`.
2. **Realistic preconditions**: PASS — `xdr.Price{N: 1, D: 20000001}` and `xdr.Price{N: 1, D: 2147483647}` are valid on-chain prices, and a failed LP-deposit still reaches the relevant conversion path.
3. **Bug vs by-design**: FAIL — this export surface intentionally follows Horizon's dual-field contract: a rounded human-readable price plus an exact rational companion. In this repository, `transactionOperationWrapper.Details()` emits `min_price` / `max_price` via `op.MinPrice.String()` and separately preserves `min_price_r` / `max_price_r`; the BigQuery-oriented path preserves the same structure, only coercing the rounded display value to `float64`. The precise value is still exported in the companion rational fields.
4. **Impact/severity**: FAIL — the hypothesis frames this as critical financial corruption, but the authoritative price is not lost. `history_operations.details` is a JSON blob, not a first-class numeric price column, and consumers already receive exact `min_price_r` / `max_price_r`. At most this is an informational representation choice for the convenience field.
5. **In scope**: PASS — if the ETL dropped the only authoritative price representation, this would be in scope. It does not.
6. **Test correctness**: PASS — the test is technically sound for demonstrating the rounded convenience field.
7. **Alternative explanations**: FAIL — the observed `0` is fully explained by the intentional `xdr.Price.String()` formatting contract rather than an accidental ETL-only corruption path.
8. **Novelty**: NOT ASSESSED — rejection does not depend on novelty.

### Rejection Reason

The PoC proves rounding of the convenience `min_price` / `max_price` field, but it does **not** prove loss of the actual bound. The ETL exports the exact value separately in `min_price_r` / `max_price_r` and mirrors Horizon's longstanding "rounded display value + exact rational" representation, so this is not a novel data-integrity bug in scope.

### Failed Checks

3, 4, 7
