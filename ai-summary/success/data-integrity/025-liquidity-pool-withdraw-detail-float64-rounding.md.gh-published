# 025: liquidity_pool_withdraw details round burned shares and realized reserves

**Date**: 2026-04-14
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

Successful `liquidity_pool_withdraw` operations export `details.shares`, `details.reserve_a_withdraw_amount`, and `details.reserve_b_withdraw_amount` by converting exact `xdr.Int64` values into `float64`. Once the scaled values are large enough, adjacent one-stroop withdrawals collapse to the same exported JSON numbers even though the underlying XDR values are distinct.

This happens on the normal `TransformOperation()` path. In the reproduced PoC, two withdraws that differ by exactly 1 stroop in burned shares and both realized reserve amounts produce identical exported values for all three fields.

## Root Cause

`TransformOperation()` copies the map returned by `extractOperationDetails()` straight into the exported `details` payload. In the `liquidity_pool_withdraw` branch, `extractOperationDetails()` takes the exact withdraw share amount from `LiquidityPoolWithdrawOp.Amount` and the exact reserve deltas from `getLiquidityPoolAndProductDelta()`, then runs all three through `utils.ConvertStroopValueToReal()`.

`ConvertStroopValueToReal()` converts the stroop integer through `big.Rat.Float64()`, which rounds to the nearest IEEE-754 `float64` before serialization. The same source file also contains a sibling liquidity-pool-withdraw formatter that uses `amount.String(...)` for exact decimal output, so the precision loss is caused by this lossy conversion choice rather than an unavoidable schema limitation.

## Reproduction

Any normal export that includes a successful `liquidity_pool_withdraw` with sufficiently large share burn and reserve deltas can hit this path. The XDR fields involved are plain `Int64` values, and the collision threshold is far below the type's maximum range.

When two adjacent values above that threshold are transformed, `extractOperationDetails()` returns identical `float64` values for `shares`, `reserve_a_withdraw_amount`, and `reserve_b_withdraw_amount`, so downstream JSON consumers cannot distinguish the distinct on-chain amounts.

## Affected Code

- `internal/transform/operation.go:TransformOperation:29-100` — copies the lossy `extractOperationDetails()` map directly into exported `details`
- `internal/transform/operation.go:extractOperationDetails:1021-1061` — stores withdraw shares and realized reserve amounts via `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:getLiquidityPoolAndProductDelta:238-281` — computes exact `xdr.Int64` reserve deltas before they are rounded away
- `internal/utils/main.go:ConvertStroopValueToReal:84-87` — converts exact stroop integers to nearest `float64`
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:28292-28297` — `LiquidityPoolWithdrawOp.Amount`, `MinAmountA`, and `MinAmountB` are native `Int64` values

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestLPWithdrawRoundsSharesAndReserves`
- **Test language**: `go`
- **How to run**: `cd <repo-root> && go build ./... && go test ./internal/transform/... -run TestLPWithdrawRoundsSharesAndReserves -count=1 -v` after creating the target file with the body below.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
	"github.com/stellar/stellar-etl/v2/internal/utils"
)

func TestLPWithdrawRoundsSharesAndReserves(t *testing.T) {
	// Two adjacent stroop values above the float64 exact-decimal threshold.
	// float64 has 53 bits of mantissa; 1-stroop precision (1e-7 XLM) is lost
	// when the XLM value exceeds ~4.5e8, i.e., stroops > ~4.5e15.
	shares1 := xdr.Int64(90071992547409930)
	shares2 := xdr.Int64(90071992547409931)

	// Large reserve deltas that also differ by 1 stroop.
	reserveDeltaA1 := xdr.Int64(90071992547409930)
	reserveDeltaA2 := xdr.Int64(90071992547409931)
	reserveDeltaB1 := xdr.Int64(90071992547409930)
	reserveDeltaB2 := xdr.Int64(90071992547409931)

	poolID := xdr.PoolId{1, 2, 3, 4, 5, 6, 7, 8, 9}

	// buildWithdrawTx constructs a successful LP-withdraw transaction whose
	// metadata encodes the given reserve deltas (pre-state reserves equal to
	// the delta, post-state reserves of zero).
	buildWithdrawTx := func(sharesAmt, deltaA, deltaB xdr.Int64) (xdr.Operation, ingest.LedgerTransaction) {
		op := xdr.Operation{
			Body: xdr.OperationBody{
				Type: xdr.OperationTypeLiquidityPoolWithdraw,
				LiquidityPoolWithdrawOp: &xdr.LiquidityPoolWithdrawOp{
					LiquidityPoolId: poolID,
					Amount:          sharesAmt,
					MinAmountA:      1,
					MinAmountB:      1,
				},
			},
		}

		successResult := utils.CreateSampleResultMeta(true, 1)

		envelope := xdr.TransactionV1Envelope{
			Tx: xdr.Transaction{
				SourceAccount: testAccount3,
				Operations:    []xdr.Operation{op},
			},
		}

		// For a withdraw: receivedX = -(postX - preX) = preX - postX.
		// Set pre = deltaX, post = 0 so receivedX = deltaX.
		lpMeta := xdr.OperationMeta{
			Changes: xdr.LedgerEntryChanges{
				{
					Type: xdr.LedgerEntryChangeTypeLedgerEntryState,
					State: &xdr.LedgerEntry{
						Data: xdr.LedgerEntryData{
							Type: xdr.LedgerEntryTypeLiquidityPool,
							LiquidityPool: &xdr.LiquidityPoolEntry{
								LiquidityPoolId: poolID,
								Body: xdr.LiquidityPoolEntryBody{
									Type: xdr.LiquidityPoolTypeLiquidityPoolConstantProduct,
									ConstantProduct: &xdr.LiquidityPoolEntryConstantProduct{
										Params: xdr.LiquidityPoolConstantProductParameters{
											AssetA: lpAssetA,
											AssetB: lpAssetB,
											Fee:    30,
										},
										ReserveA:        deltaA,
										ReserveB:        deltaB,
										TotalPoolShares: sharesAmt + 1000,
									},
								},
							},
						},
					},
				},
				{
					Type: xdr.LedgerEntryChangeTypeLedgerEntryUpdated,
					Updated: &xdr.LedgerEntry{
						Data: xdr.LedgerEntryData{
							Type: xdr.LedgerEntryTypeLiquidityPool,
							LiquidityPool: &xdr.LiquidityPoolEntry{
								LiquidityPoolId: poolID,
								Body: xdr.LiquidityPoolEntryBody{
									Type: xdr.LiquidityPoolTypeLiquidityPoolConstantProduct,
									ConstantProduct: &xdr.LiquidityPoolEntryConstantProduct{
										Params: xdr.LiquidityPoolConstantProductParameters{
											AssetA: lpAssetA,
											AssetB: lpAssetB,
											Fee:    30,
										},
										ReserveA:        0,
										ReserveB:        0,
										TotalPoolShares: 1000,
									},
								},
							},
						},
					},
				},
			},
		}

		tx := ingest.LedgerTransaction{
			Index: 1,
			Envelope: xdr.TransactionEnvelope{
				Type: xdr.EnvelopeTypeEnvelopeTypeTx,
				V1:   &envelope,
			},
			Result: successResult.Result,
			UnsafeMeta: xdr.TransactionMeta{
				V: 1,
				V1: &xdr.TransactionMetaV1{
					Operations: []xdr.OperationMeta{lpMeta},
				},
			},
		}
		return op, tx
	}

	op1, tx1 := buildWithdrawTx(shares1, reserveDeltaA1, reserveDeltaB1)
	op2, tx2 := buildWithdrawTx(shares2, reserveDeltaA2, reserveDeltaB2)

	details1, err := extractOperationDetails(op1, tx1, 0, "Test SDF Network ; September 2015")
	if err != nil {
		t.Fatalf("extractOperationDetails op1: %v", err)
	}
	details2, err := extractOperationDetails(op2, tx2, 0, "Test SDF Network ; September 2015")
	if err != nil {
		t.Fatalf("extractOperationDetails op2: %v", err)
	}

	// --- shares collision ---
	sharesOut1 := details1["shares"].(float64)
	sharesOut2 := details2["shares"].(float64)
	if shares1 == shares2 {
		t.Fatal("test setup error: shares inputs are identical")
	}
	if sharesOut1 != sharesOut2 {
		t.Fatalf("expected shares float64 collision, got different values: %.20g vs %.20g", sharesOut1, sharesOut2)
	}
	t.Logf("SHARES COLLISION: stroops %d and %d both export as %.20g", shares1, shares2, sharesOut1)

	// --- reserve_a_withdraw_amount collision ---
	resAOut1 := details1["reserve_a_withdraw_amount"].(float64)
	resAOut2 := details2["reserve_a_withdraw_amount"].(float64)
	if reserveDeltaA1 == reserveDeltaA2 {
		t.Fatal("test setup error: reserveA inputs are identical")
	}
	if resAOut1 != resAOut2 {
		t.Fatalf("expected reserve_a_withdraw_amount collision, got different values: %.20g vs %.20g", resAOut1, resAOut2)
	}
	t.Logf("RESERVE_A COLLISION: stroops %d and %d both export as %.20g", reserveDeltaA1, reserveDeltaA2, resAOut1)

	// --- reserve_b_withdraw_amount collision ---
	resBOut1 := details1["reserve_b_withdraw_amount"].(float64)
	resBOut2 := details2["reserve_b_withdraw_amount"].(float64)
	if reserveDeltaB1 == reserveDeltaB2 {
		t.Fatal("test setup error: reserveB inputs are identical")
	}
	if resBOut1 != resBOut2 {
		t.Fatalf("expected reserve_b_withdraw_amount collision, got different values: %.20g vs %.20g", resBOut1, resBOut2)
	}
	t.Logf("RESERVE_B COLLISION: stroops %d and %d both export as %.20g", reserveDeltaB1, reserveDeltaB2, resBOut1)

	t.Log("BUG CONFIRMED: LP withdraw shares and reserve amounts lose precision via float64 rounding")
}
```

## Expected vs Actual Behavior

- **Expected**: Distinct on-chain burn-share and realized-reserve amounts should remain distinguishable in exported operation details, e.g. `9007199254.7409930` vs `9007199254.7409931`.
- **Actual**: Adjacent values export as the same floating-point numbers for `shares`, `reserve_a_withdraw_amount`, and `reserve_b_withdraw_amount`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC constructs successful withdraw metadata so the production `extractOperationDetails()` liquidity-pool-withdraw branch computes exact reserve deltas and then exports the lossy fields.
2. Realistic preconditions: YES — the affected XDR fields are ordinary `Int64` amounts, and the collision threshold is far below their allowed range.
3. Bug vs by-design: BUG — a sibling liquidity-pool-withdraw formatter in the same package already uses exact `amount.String(...)` output, so this loss of precision is not required by the surrounding model.
4. Final severity: Critical — these are monetary fields in normal operation exports, so downstream accounting and reconciliation can read plausible but wrong values.
5. In scope: YES — this is concrete financial data corruption in repository-owned ETL logic.
6. Test correctness: CORRECT — the test uses real XDR operation and metadata types, feeds the live withdraw-details branch, proves the source values differ by 1 stroop, and shows all three exported fields collide.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Export liquidity-pool-withdraw share burns and realized reserves through an exact representation instead of `float64`, preferably reusing `amount.String(...)` or a shared exact-decimal helper for operation details. If a numeric approximation must remain for compatibility, add a separate exact field so the rounded float is not the only exported value.
