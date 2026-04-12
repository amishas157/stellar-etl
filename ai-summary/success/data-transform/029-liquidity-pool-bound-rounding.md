# 029: Liquidity-pool request bounds round large reserve limits

**Date**: 2026-04-12
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformOperation()` exports liquidity-pool deposit and withdraw request bounds through `float64`, so distinct on-chain reserve limits can collapse to the same JSON number in `history_operations.details`. For sufficiently large `MaxAmount*` / `MinAmount*` values, a 1-stroop difference disappears even though the same package already has exact decimal-string formatting for these fields.

## Root Cause

The `LiquidityPoolDeposit` and `LiquidityPoolWithdraw` branches of `extractOperationDetails()` send `MaxAmountA`, `MaxAmountB`, `MinAmountA`, and `MinAmountB` through `utils.ConvertStroopValueToReal()`. That helper converts the exact `xdr.Int64` value to the nearest `float64`, which cannot preserve 7-decimal stroop precision once the scaled decimal exceeds float64's exact range. `TransformOperation()` then copies that lossy map into both `details` and `details_json`, so the rounded value is what gets exported.

## Reproduction

Create two otherwise identical liquidity-pool operations whose request-side reserve bounds differ by 1 stroop at `90071992547409930` and `90071992547409931`. Run them through the production operation-detail path and compare `reserve_a_max_amount` or `reserve_a_min_amount`: both export as the same `float64` even though `amount.String(...)` still distinguishes `9007199254.7409930` from `9007199254.7409931`.

## Affected Code

- `internal/transform/operation.go:957-1019` — liquidity-pool deposit details export `reserve_a_max_amount` / `reserve_b_max_amount` via `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:1021-1061` — liquidity-pool withdraw details export `reserve_a_min_amount` / `reserve_b_min_amount` via the same lossy helper
- `internal/utils/main.go:84-87` — `ConvertStroopValueToReal()` returns the nearest `float64` for an `xdr.Int64` stroop amount
- `internal/transform/operation.go:54-57,85-98` — `TransformOperation()` forwards the lossy detail map directly into both exported detail fields
- `internal/transform/operation.go:1603-1684` — sibling liquidity-pool formatter uses exact `amount.String(...)` values for the same logical bounds

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestLiquidityPoolBoundRoundingPrecisionLoss`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then run `go build ./... && go test ./internal/transform/... -run TestLiquidityPoolBoundRoundingPrecisionLoss -v`.

### Test Body

```go
package transform

import (
	"math/big"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/utils"

	"github.com/stellar/go-stellar-sdk/amount"
	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestLiquidityPoolBoundRoundingPrecisionLoss demonstrates that
// ConvertStroopValueToReal loses low-order stroop precision for large
// values, causing distinct on-chain reserve bounds to collapse to the
// same float64 in the exported operation details.
func TestLiquidityPoolBoundRoundingPrecisionLoss(t *testing.T) {
	// Two stroop values differing by 1 that map to the same float64.
	// 2^53 = 9007199254740992; these values exceed that threshold when
	// divided by 1e7, so the fractional part loses the last digit.
	stroopA := xdr.Int64(90071992547409930)
	stroopB := xdr.Int64(90071992547409931)

	// --- Part 1: Direct demonstration of ConvertStroopValueToReal precision loss ---
	floatA := utils.ConvertStroopValueToReal(stroopA)
	floatB := utils.ConvertStroopValueToReal(stroopB)

	if floatA != floatB {
		t.Fatalf("Expected ConvertStroopValueToReal to collapse distinct values, "+
			"but got different float64s: %v vs %v", floatA, floatB)
	}

	// Confirm the exact rational values are actually distinct
	ratA, _ := big.NewRat(int64(stroopA), 10000000).Float64()
	ratB, _ := big.NewRat(int64(stroopB), 10000000).Float64()
	t.Logf("ConvertStroopValueToReal precision loss confirmed:")
	t.Logf("  stroopA = %d -> float64 = %.20f", stroopA, ratA)
	t.Logf("  stroopB = %d -> float64 = %.20f", stroopB, ratB)
	t.Logf("  float64 values equal: %v (precision lost)", ratA == ratB)

	// Show that amount.String preserves the distinction
	exactA := amount.String(stroopA)
	exactB := amount.String(stroopB)
	if exactA == exactB {
		t.Fatalf("amount.String should distinguish the two values, but got %s for both", exactA)
	}
	t.Logf("  amount.String(stroopA) = %s", exactA)
	t.Logf("  amount.String(stroopB) = %s", exactB)
	t.Logf("  amount.String values distinct: true (exact representation preserved)")

	// --- Part 2: Demonstrate via extractOperationDetails for LiquidityPoolDeposit ---
	depositOpA := xdr.Operation{
		SourceAccount: &genericSourceAccount,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeLiquidityPoolDeposit,
			LiquidityPoolDepositOp: &xdr.LiquidityPoolDepositOp{
				LiquidityPoolId: xdr.PoolId{1, 2, 3},
				MaxAmountA:      stroopA,
				MaxAmountB:      1000,
				MinPrice:        xdr.Price{N: 1, D: 1},
				MaxPrice:        xdr.Price{N: 1, D: 1},
			},
		},
	}
	depositOpB := xdr.Operation{
		SourceAccount: &genericSourceAccount,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeLiquidityPoolDeposit,
			LiquidityPoolDepositOp: &xdr.LiquidityPoolDepositOp{
				LiquidityPoolId: xdr.PoolId{1, 2, 3},
				MaxAmountA:      stroopB,
				MaxAmountB:      1000,
				MinPrice:        xdr.Price{N: 1, D: 1},
				MaxPrice:        xdr.Price{N: 1, D: 1},
			},
		},
	}

	// Use a failed transaction to avoid needing LP ledger changes. The request-side
	// bounds are still exported on the same production path regardless of success.
	failedTx := ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1: &xdr.TransactionV1Envelope{
				Tx: xdr.Transaction{
					SourceAccount: genericSourceAccount,
					Operations:    []xdr.Operation{depositOpA},
				},
			},
		},
		Result: utils.CreateSampleResultMeta(false, 1).Result,
		UnsafeMeta: xdr.TransactionMeta{
			V: 1,
			V1: &xdr.TransactionMetaV1{
				Operations: []xdr.OperationMeta{{}},
			},
		},
	}

	detailsA, err := extractOperationDetails(depositOpA, failedTx, 0, "test")
	if err != nil {
		t.Fatalf("extractOperationDetails for deposit A failed: %v", err)
	}

	failedTx.Envelope.V1.Tx.Operations = []xdr.Operation{depositOpB}
	detailsB, err := extractOperationDetails(depositOpB, failedTx, 0, "test")
	if err != nil {
		t.Fatalf("extractOperationDetails for deposit B failed: %v", err)
	}

	depositMaxA := detailsA["reserve_a_max_amount"].(float64)
	depositMaxB := detailsB["reserve_a_max_amount"].(float64)

	if depositMaxA != depositMaxB {
		t.Fatalf("Expected reserve_a_max_amount to lose precision, "+
			"but got different values: %v vs %v", depositMaxA, depositMaxB)
	}
	t.Logf("LiquidityPoolDeposit reserve_a_max_amount precision loss confirmed:")
	t.Logf("  Input MaxAmountA: %d vs %d (differ by 1 stroop)", stroopA, stroopB)
	t.Logf("  Output reserve_a_max_amount: %.20f vs %.20f (identical)", depositMaxA, depositMaxB)

	// --- Part 3: Demonstrate via extractOperationDetails for LiquidityPoolWithdraw ---
	withdrawOpA := xdr.Operation{
		SourceAccount: &genericSourceAccount,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeLiquidityPoolWithdraw,
			LiquidityPoolWithdrawOp: &xdr.LiquidityPoolWithdrawOp{
				LiquidityPoolId: xdr.PoolId{1, 2, 3},
				Amount:          1000,
				MinAmountA:      stroopA,
				MinAmountB:      1000,
			},
		},
	}
	withdrawOpB := xdr.Operation{
		SourceAccount: &genericSourceAccount,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeLiquidityPoolWithdraw,
			LiquidityPoolWithdrawOp: &xdr.LiquidityPoolWithdrawOp{
				LiquidityPoolId: xdr.PoolId{1, 2, 3},
				Amount:          1000,
				MinAmountA:      stroopB,
				MinAmountB:      1000,
			},
		},
	}

	failedTx.Envelope.V1.Tx.Operations = []xdr.Operation{withdrawOpA}
	detailsWA, err := extractOperationDetails(withdrawOpA, failedTx, 0, "test")
	if err != nil {
		t.Fatalf("extractOperationDetails for withdraw A failed: %v", err)
	}

	failedTx.Envelope.V1.Tx.Operations = []xdr.Operation{withdrawOpB}
	detailsWB, err := extractOperationDetails(withdrawOpB, failedTx, 0, "test")
	if err != nil {
		t.Fatalf("extractOperationDetails for withdraw B failed: %v", err)
	}

	withdrawMinA := detailsWA["reserve_a_min_amount"].(float64)
	withdrawMinB := detailsWB["reserve_a_min_amount"].(float64)

	if withdrawMinA != withdrawMinB {
		t.Fatalf("Expected reserve_a_min_amount to lose precision, "+
			"but got different values: %v vs %v", withdrawMinA, withdrawMinB)
	}
	t.Logf("LiquidityPoolWithdraw reserve_a_min_amount precision loss confirmed:")
	t.Logf("  Input MinAmountA: %d vs %d (differ by 1 stroop)", stroopA, stroopB)
	t.Logf("  Output reserve_a_min_amount: %.20f vs %.20f (identical)", withdrawMinA, withdrawMinB)

	// Summary
	t.Logf("\n=== SUMMARY ===")
	t.Logf("Two distinct stroop values (%d and %d) produce identical float64 output", stroopA, stroopB)
	t.Logf("in reserve_a_max_amount (deposit) and reserve_a_min_amount (withdraw).")
	t.Logf("The sibling formatter in the same file uses amount.String() which preserves")
	t.Logf("exact values: %s vs %s", exactA, exactB)
}
```

## Expected vs Actual Behavior

- **Expected**: liquidity-pool request bounds in exported operation details preserve distinct 7-decimal on-chain values, so adjacent reserve limits remain distinguishable
- **Actual**: `reserve_a_max_amount`, `reserve_b_max_amount`, `reserve_a_min_amount`, and `reserve_b_min_amount` round through `float64`, so different on-chain bounds can export as the same plausible JSON number

## Adversarial Review

1. Exercises claimed bug: YES — the test drives the exact `LiquidityPoolDeposit` / `LiquidityPoolWithdraw` production branches that populate these fields, and `TransformOperation()` forwards that map unchanged into both exported detail surfaces.
2. Realistic preconditions: YES — these request-bound fields are `xdr.Int64` amounts and can represent large issued-asset quantities; the trigger does not rely on impossible XDR values or private-only APIs.
3. Bug vs by-design: BUG — the transform silently changes exact on-chain bounds into lossy floats even though the same package already has exact `amount.String(...)` formatting for the same logical values.
4. Final severity: Critical — these are exported monetary constraint fields, so downstream analytics and reconciliation can read a wrong numeric value without any error signal.
5. In scope: YES — this is a concrete transform-layer data corruption issue in normal export output.
6. Test correctness: CORRECT — the assertions compare two distinct valid on-chain amounts and show they collapse only after the production conversion step, not because of circular setup.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Stop exporting these request-bound liquidity-pool amounts as `float64`. Preserve the exact decimal strings from `amount.String(...)`, matching the sibling liquidity-pool formatter, or use an exact decimal type rather than `float64` for exported monetary fields.
