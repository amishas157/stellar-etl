# H002: `liquidity_pool_withdraw` rounds share burn and realized reserve amounts

**Date**: 2026-04-14
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

A successful `liquidity_pool_withdraw` row should preserve the exact burned share amount and exact withdrawn reserve amounts derived from the ledger delta. If two valid withdrawals differ by 1 stroop in `shares` or in either realized reserve amount, the exported `details.shares`, `details.reserve_a_withdraw_amount`, and `details.reserve_b_withdraw_amount` should remain different.

## Mechanism

The live withdraw branch converts `op.Amount`, `receivedA`, and `receivedB` through `utils.ConvertStroopValueToReal()`, so large successful withdrawals are rounded to the nearest `float64`. The duplicate formatter in the same file keeps these same values exact with `amount.String(...)`, which means the live exporter is discarding precision that is still available after `getLiquidityPoolAndProductDelta()` computes the exact `Int64` reserve deltas.

## Trigger

Export two otherwise identical successful `liquidity_pool_withdraw` operations whose burned shares or realized reserve deltas differ by 1 stroop at magnitudes above float64's exact decimal precision, e.g. a `shares` value of `90071992547409930` versus `90071992547409931`. Compare `details["shares"]` or either `reserve_*_withdraw_amount`: the current exporter should collapse the rows to the same numeric output.

## Target Code

- `internal/transform/operation.go:1047-1061` — live withdraw branch writes `reserve_*_withdraw_amount` and `shares` via `ConvertStroopValueToReal`
- `internal/transform/operation.go:1676-1682` — sibling formatter preserves `shares` and realized reserves with `amount.String(...)`
- `internal/utils/main.go:84-87` — stroop conversion ends in `float64`

## Evidence

The live path computes exact reserve deltas from the liquidity-pool ledger change and then immediately rounds them when populating operation details. The duplicate formatter proves the same data can be carried forward losslessly as decimal strings, so this is not an unavoidable consequence of the upstream XDR.

## Anti-Evidence

The already-published LP bound findings cover `reserve_*_min_amount`, not the realized withdraw amounts or burned shares targeted here. Small withdrawals still serialize correctly; the bug requires large values whose 7-decimal representation exceeds float64 precision.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-14
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated in fail/ or success/

### Trace Summary

Traced the LP withdraw branch in `extractOperationDetails()` (operation.go:1021-1061). The function obtains exact `xdr.Int64` reserve deltas from `getLiquidityPoolAndProductDelta()` and the exact `op.Amount` shares, then immediately converts all three to `float64` via `utils.ConvertStroopValueToReal()`. This function (utils/main.go:85-87) uses `big.NewRat(int64(input), 10000000).Float64()` — a single rounding step that loses precision for values above ~4.5×10^15 stroops (~450M XLM). There are no companion fields preserving the exact integer values. The dead-code sibling formatter (`transactionOperationWrapper.Details()` at line 1676-1684) uses `amount.String()` for the same data, proving lossless representation is available but unused in the production path.

### Code Paths Examined

- `internal/transform/operation.go:30-100` — `TransformOperation()` calls `extractOperationDetails()` at line 54, stores the resulting `map[string]interface{}` as `OperationOutput.OperationDetails`
- `internal/transform/operation.go:1021-1061` — `extractOperationDetails()` LP withdraw case: obtains exact `xdr.Int64` deltas from `getLiquidityPoolAndProductDelta()` then stores `details["reserve_a_withdraw_amount"]`, `details["reserve_b_withdraw_amount"]`, and `details["shares"]` all as `float64` via `ConvertStroopValueToReal()`
- `internal/utils/main.go:84-87` — `ConvertStroopValueToReal(input xdr.Int64) float64` returns `big.NewRat(int64(input), 10000000).Float64()` — confirmed single-step float64 rounding
- `internal/transform/operation.go:1645-1684` — `transactionOperationWrapper.Details()` LP withdraw case: uses `amount.String(op.Amount)` and `amount.String(receivedA/B)` for lossless string representation — dead code, never called in production
- `internal/transform/schema.go:137-149` — `OperationOutput.OperationDetails` is `map[string]interface{}` — no typed companion fields for amounts; the float64 is the only exported value
- `internal/transform/operation.go:238-285` — `getLiquidityPoolAndProductDelta()` computes exact `xdr.Int64` reserve deltas from liquidity pool ledger entry changes — precision is available upstream

### Findings

1. **Confirmed precision loss**: `ConvertStroopValueToReal` returns a `float64`. The ULP (unit in last place) at value V is approximately `V × 2^-52`. For 1-stroop precision (1e-7 in scaled XLM), we need `V × 2^-52 < 1e-7`, giving threshold `V < 1e-7 × 2^52 ≈ 4.5 × 10^8`. Above ~450M XLM (~4.5×10^15 stroops), adjacent stroop values collapse to the same float64.

2. **No companion fields**: Unlike the LP deposit price case (which was rejected because exact rationals survive in `*_price_r`), the LP withdraw amounts have NO companion field. The float64 is the sole exported representation for `shares`, `reserve_a_withdraw_amount`, and `reserve_b_withdraw_amount`.

3. **Dead code proves lossless path exists**: `transactionOperationWrapper.Details()` (line 1680) uses `amount.String(op.Amount)` and (line 1681-1683) `amount.String(receivedA/B)` for exact decimal strings. This code is never called — `Details()` has zero call sites in the production codebase — but demonstrates the exact representation is trivially available.

4. **Systemic pattern**: `ConvertStroopValueToReal` is used for 30+ financial fields across `operation.go`, `account.go`, `trustline.go`, `offer.go`, `trade.go`, and `liquidity_pool.go`. The LP withdraw amounts are one instance of a codebase-wide pattern. An in-flight finding for `change_trust` limit (poc/001) covers the same underlying function.

5. **Distinct from success/002**: The token transfer finding (success/002) covers `strconv.ParseFloat` double-rounding in `token_transfer.go`. This hypothesis targets `ConvertStroopValueToReal` single-rounding in `operation.go` — different code path, different conversion mechanism, different export table.

### PoC Guidance

- **Test file**: `internal/transform/data_integrity_poc_test.go` (or `internal/transform/operation_test.go`)
- **Setup**: Create two `LiquidityPoolWithdrawOp` operations with `Amount` (shares) values of `90071992547409930` and `90071992547409931` (xdr.Int64). Each must be wrapped in a successful `ingest.LedgerTransaction` with LP metadata that includes a `LiquidityPoolEntry` in both pre and post states so `getLiquidityPoolAndProductDelta()` can compute deltas. Set up the LP reserve changes so that `receivedA` and `receivedB` also exceed the precision threshold.
- **Steps**: Call `TransformOperation()` for each operation. Extract `details["shares"]`, `details["reserve_a_withdraw_amount"]`, and `details["reserve_b_withdraw_amount"]` from both outputs.
- **Assertion**: Assert that the two `details["shares"]` float64 values are identical despite distinct stroop inputs (demonstrating the collision). Separately, assert that `amount.String()` on the same Int64 values produces distinct strings, confirming the exact representation is available. Optionally, verify the same collision for `reserve_a_withdraw_amount` and `reserve_b_withdraw_amount` using LP metadata with large reserve deltas differing by 1 stroop.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-14
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestLPWithdrawRoundsSharesAndReserves"
**Test Language**: Go

### Demonstration

The test constructs two successful `liquidity_pool_withdraw` operations with share amounts and reserve deltas differing by exactly 1 stroop at magnitude ~9×10^16. After running through the production `extractOperationDetails()` code path, all three financial fields — `shares`, `reserve_a_withdraw_amount`, and `reserve_b_withdraw_amount` — collapse to identical float64 values (9007199254.7409934998) despite distinct integer inputs. This confirms that `ConvertStroopValueToReal()` discards precision for amounts above ~4.5×10^15 stroops, and no companion field preserves the exact value.

### Test Body

```go
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

### Test Output

```
=== RUN   TestLPWithdrawRoundsSharesAndReserves
    data_integrity_poc_test.go:305: SHARES COLLISION: stroops 90071992547409930 and 90071992547409931 both export as 9007199254.7409934998
    data_integrity_poc_test.go:316: RESERVE_A COLLISION: stroops 90071992547409930 and 90071992547409931 both export as 9007199254.7409934998
    data_integrity_poc_test.go:327: RESERVE_B COLLISION: stroops 90071992547409930 and 90071992547409931 both export as 9007199254.7409934998
    data_integrity_poc_test.go:329: BUG CONFIRMED: LP withdraw shares and reserve amounts lose precision via float64 rounding
--- PASS: TestLPWithdrawRoundsSharesAndReserves (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.649s
```
