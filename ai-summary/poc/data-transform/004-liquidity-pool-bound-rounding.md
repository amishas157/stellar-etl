# H004: Liquidity-pool request bounds round large reserve limits in operation details

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `liquidity_pool_deposit` and `liquidity_pool_withdraw`, the request-side reserve bounds in `history_operations.details` should preserve the exact on-chain stroop values. Fields such as `reserve_a_max_amount`, `reserve_b_max_amount`, `reserve_a_min_amount`, and `reserve_b_min_amount` should remain distinct when the underlying XDR values differ by 1 stroop.

## Mechanism

The live deposit and withdraw branches convert these bound fields through `utils.ConvertStroopValueToReal()`, so large values lose low-order stroop precision before export. The same file's alternate formatter already emits the corresponding reserve bounds with exact `amount.String(...)` values inside `reserves_max` / `reserves_min`, which shows this decoded JSON surface can carry exact decimal strings without changing the schema type of `OperationDetails` itself.

## Trigger

1. Build a successful `liquidity_pool_deposit` with `MaxAmountA` or `MaxAmountB` equal to `90071992547409930` stroops and a second one with `90071992547409931`.
2. Build a successful `liquidity_pool_withdraw` with `MinAmountA` or `MinAmountB` differing by 1 stroop at the same magnitude.
3. Run them through `TransformOperation()` and compare the exported reserve bound fields.

## Target Code

- `internal/transform/operation.go:957-1013` — deposit detail export converts `reserve_a_max_amount` / `reserve_b_max_amount` with `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:1021-1060` — withdraw detail export converts `reserve_a_min_amount` / `reserve_b_min_amount` with `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:1603-1684` — sibling formatter keeps exact bound values via `amount.String(...)`

## Evidence

The live output uses exact decimal strings for some adjacent liquidity-pool detail fields (`shares` in withdraw, `destination_min` elsewhere in the file) but still writes the request-side reserve bounds as `float64`. The alternate formatter preserves those same bounds exactly in the same source file, so large bound values can silently collapse only because the active branch chooses the lossy path.

## Anti-Evidence

The realized reserve deltas and share outputs are separate fields, and the envelope XDR can still be decoded elsewhere. But downstream users reading `history_operations.details` as the decoded request payload will see incorrect reserve constraints for large LP operations.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the `extractOperationDetails()` function for both `LiquidityPoolDeposit` (lines 957-1019) and `LiquidityPoolWithdraw` (lines 1021-1062) branches. Confirmed that `reserve_a_max_amount` (line 990), `reserve_b_max_amount` (line 1001), `reserve_a_min_amount` (line 1051), and `reserve_b_min_amount` (line 1058) are all converted via `utils.ConvertStroopValueToReal()`, which calls `big.NewRat(int64(input), 10000000).Float64()` — returning the nearest float64 representation. The sibling formatter at lines 1631-1634 and 1676-1679 uses exact `amount.String()` for the same logical values, confirming the lossy path is unnecessary.

### Code Paths Examined

- `internal/transform/operation.go:990` — `details["reserve_a_max_amount"] = utils.ConvertStroopValueToReal(op.MaxAmountA)` — lossy float64 conversion of deposit max bound
- `internal/transform/operation.go:1001` — `details["reserve_b_max_amount"] = utils.ConvertStroopValueToReal(op.MaxAmountB)` — lossy float64 conversion of deposit max bound
- `internal/transform/operation.go:1051` — `details["reserve_a_min_amount"] = utils.ConvertStroopValueToReal(op.MinAmountA)` — lossy float64 conversion of withdraw min bound
- `internal/transform/operation.go:1058` — `details["reserve_b_min_amount"] = utils.ConvertStroopValueToReal(op.MinAmountB)` — lossy float64 conversion of withdraw min bound
- `internal/utils/main.go:ConvertStroopValueToReal` — `big.NewRat(int64(input), 10000000).Float64()` returns nearest float64
- `internal/transform/operation.go:1631-1634` — sibling formatter uses `amount.String(op.MaxAmountA)` for exact deposit bounds
- `internal/transform/operation.go:1676-1679` — sibling formatter uses `amount.String(op.MinAmountA)` for exact withdraw bounds

### Findings

The four request-side reserve bound fields (`reserve_a_max_amount`, `reserve_b_max_amount`, `reserve_a_min_amount`, `reserve_b_min_amount`) are distinct from the realized delta fields covered by success/017. These fields represent the depositor's maximum willingness or the withdrawer's minimum acceptable amounts — monetary constraint values that downstream systems use to understand transaction intent. The `ConvertStroopValueToReal()` conversion loses precision for values above ~9007199254 XLM (2^53 stroops), causing adjacent 1-stroop-different bounds to collapse to the same float64. The sibling formatter in the same file proves exact decimal output was available and intended.

Additionally, the withdraw branch has three more fields using the same lossy conversion: `reserve_a_withdraw_amount` (line 1052), `reserve_b_withdraw_amount` (line 1059), and `shares` (line 1061) — these are also affected but the hypothesis correctly focuses on the request-side bounds as the primary novel finding.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go` (or a new `data_integrity_poc_test.go` in the same package)
- **Setup**: Construct two `LiquidityPoolDepositOp` operations with `MaxAmountA` values of `90071992547409930` and `90071992547409931` stroops. Similarly construct two `LiquidityPoolWithdrawOp` operations with `MinAmountA` values differing by 1 stroop at the same magnitude. Wrap in valid `ingest.LedgerTransaction` with successful results and LP ledger changes.
- **Steps**: Call `extractOperationDetails()` or `TransformOperation()` for each pair.
- **Assertion**: Assert that `details["reserve_a_max_amount"]` (deposit) and `details["reserve_a_min_amount"]` (withdraw) produce the same float64 for two inputs that differ by 1 stroop, demonstrating the precision loss. Compare against `amount.String()` output to show the exact values are distinct.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestLiquidityPoolBoundRoundingPrecisionLoss"
**Test Language**: Go

### Demonstration

The test constructs two `LiquidityPoolDepositOp` operations with `MaxAmountA` values of 90071992547409930 and 90071992547409931 stroops (differing by 1 stroop), and two `LiquidityPoolWithdrawOp` operations with `MinAmountA` values at the same magnitudes. Running each pair through `extractOperationDetails()` produces identical float64 values for `reserve_a_max_amount` and `reserve_a_min_amount` respectively, proving that distinct on-chain reserve bounds collapse to the same output value. Meanwhile, `amount.String()` correctly distinguishes "9007199254.7409930" from "9007199254.7409931", confirming that exact representation is available but unused in the active code path.

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

	// Use a failed transaction to avoid needing LP ledger changes
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

### Test Output

```
=== RUN   TestLiquidityPoolBoundRoundingPrecisionLoss
    data_integrity_poc_test.go:37: ConvertStroopValueToReal precision loss confirmed:
    data_integrity_poc_test.go:38:   stroopA = 90071992547409930 -> float64 = 9007199254.74099349975585937500
    data_integrity_poc_test.go:39:   stroopB = 90071992547409931 -> float64 = 9007199254.74099349975585937500
    data_integrity_poc_test.go:40:   float64 values equal: true (precision lost)
    data_integrity_poc_test.go:48:   amount.String(stroopA) = 9007199254.7409930
    data_integrity_poc_test.go:49:   amount.String(stroopB) = 9007199254.7409931
    data_integrity_poc_test.go:50:   amount.String values distinct: true (exact representation preserved)
    data_integrity_poc_test.go:119: LiquidityPoolDeposit reserve_a_max_amount precision loss confirmed:
    data_integrity_poc_test.go:120:   Input MaxAmountA: 90071992547409930 vs 90071992547409931 (differ by 1 stroop)
    data_integrity_poc_test.go:121:   Output reserve_a_max_amount: 9007199254.74099349975585937500 vs 9007199254.74099349975585937500 (identical)
    data_integrity_poc_test.go:168: LiquidityPoolWithdraw reserve_a_min_amount precision loss confirmed:
    data_integrity_poc_test.go:169:   Input MinAmountA: 90071992547409930 vs 90071992547409931 (differ by 1 stroop)
    data_integrity_poc_test.go:170:   Output reserve_a_min_amount: 9007199254.74099349975585937500 vs 9007199254.74099349975585937500 (identical)
    data_integrity_poc_test.go:173:
        === SUMMARY ===
    data_integrity_poc_test.go:174: Two distinct stroop values (90071992547409930 and 90071992547409931) produce identical float64 output
    data_integrity_poc_test.go:175: in reserve_a_max_amount (deposit) and reserve_a_min_amount (withdraw).
    data_integrity_poc_test.go:176: The sibling formatter in the same file uses amount.String() which preserves
    data_integrity_poc_test.go:177: exact values: 9007199254.7409930 vs 9007199254.7409931
--- PASS: TestLiquidityPoolBoundRoundingPrecisionLoss (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.681s
```
