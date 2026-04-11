# H002: Liquidity-pool deposit details round large reserve deltas and share amounts

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a successful `liquidity_pool_deposit` operation, the exported detail fields `reserve_a_deposit_amount`, `reserve_b_deposit_amount`, and `shares_received` should preserve the exact decimal values implied by the underlying `xdr.Int64` reserve/share deltas. A one-stroop difference in the pool deltas should remain visible in the operation JSON.

## Mechanism

The deposit detail builder first formats the exact `xdr.Int64` amounts as decimal strings with `amount.String(...)`, then immediately parses those strings back into `float64` with `strconv.ParseFloat(..., 64)`. For large deltas, the float64 parse rounds away low-order digits, so operation details publish plausible-but-wrong deposit and share values even though the preceding string conversion had the exact decimal representation available.

## Trigger

Export a successful `liquidity_pool_deposit` whose reserve delta or `TotalPoolShares` delta exceeds float64's exact-integer range after 7-decimal scaling, such as a deposit delta of `9007199254740993` stroops. The detail map will report `reserve_*_deposit_amount` or `shares_received` as the rounded neighbor value instead of the exact amount.

## Target Code

- `internal/transform/operation.go:974-1019` — successful deposit path computes deltas, then parses `amount.String(...)` back into `float64`
- `internal/transform/operation.go:238-285` — `getLiquidityPoolAndProductDelta()` returns exact `xdr.Int64` reserve/share deltas before the rounding step
- `internal/transform/operation_test.go:1751-1804` — tests cover only tiny deposit/share values

## Evidence

The code already has the exact decimal strings in hand; the only reason precision is lost is the final `ParseFloat` step. This is a live transform-layer bug in the JSON operation export path, not a Parquet-only issue, and it affects three separate monetary fields on the same operation row.

## Anti-Evidence

Typical test fixtures use tiny reserve deltas, so the exported values look correct in current coverage. If downstream consumers ignore these float fields and rebuild amounts from raw XDR elsewhere, they can recover, but the exported operation details themselves are still corrupted.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `TransformOperation` → `extractOperationDetails` → `LiquidityPoolDeposit` case (lines 957-1019). Confirmed that `getLiquidityPoolAndProductDelta` returns exact `xdr.Int64` deltas, `amount.String()` converts them to exact 7-decimal strings via `big.Rat.FloatString(7)`, and then `strconv.ParseFloat(..., 64)` immediately parses back to float64, discarding precision for values beyond 2^53. Three financial fields (`reserve_a_deposit_amount`, `reserve_b_deposit_amount`, `shares_received`) are affected. Notably, the sibling `transactionOperationWrapper.Details()` method (line 1603-1649) stores the same amounts as exact strings, confirming the string representation was available and the ParseFloat step is gratuitous.

### Code Paths Examined

- `internal/transform/operation.go:TransformOperation:30-101` — confirmed it calls `extractOperationDetails` at line 54, producing the exported `OperationDetails`
- `internal/transform/operation.go:extractOperationDetails:957-1019` — confirmed the `LiquidityPoolDeposit` case uses `strconv.ParseFloat(amount.String(depositedA), 64)` on lines 991, 1002, 1015 for all three financial fields
- `internal/transform/operation.go:getLiquidityPoolAndProductDelta:238-285` — confirmed it returns exact `xdr.Int64` deltas computed as `postA - preA`
- `github.com/stellar/go-stellar-sdk/amount/main.go:String:134-136` → `StringFromInt64:155-159` — confirmed `amount.String()` uses `big.Rat.FloatString(7)` which produces an exact decimal string
- `internal/transform/operation.go:transactionOperationWrapper.Details:1603-1649` — confirmed the alternative code path stores the same amounts as strings via `amount.String()` without ParseFloat roundtrip
- `internal/utils/main.go:ConvertStroopValueToReal` — confirmed line 990 uses this for `reserve_a_max_amount` while lines 991-995 use the ParseFloat pattern for `reserve_a_deposit_amount`, showing inconsistent conversion methods for adjacent financial fields

### Findings

1. **Confirmed mechanism**: `amount.String(depositedA)` produces the exact string `"900719925.4740993"` for 9007199254740993 stroops. `strconv.ParseFloat("900719925.4740993", 64)` rounds this to `900719925.4740992` because float64 cannot represent 16 significant digits exactly. The rounded value is stored in the detail map.

2. **Three affected fields**: `reserve_a_deposit_amount` (line 995), `reserve_b_deposit_amount` (line 1006), `shares_received` (line 1019) all use the identical lossy pattern.

3. **Inconsistency with sibling code path**: The `transactionOperationWrapper.Details()` method (lines 1645-1649) stores the same data as exact strings. This proves the exact representation was architecturally available.

4. **Inconsistency within same function**: `reserve_a_max_amount` (line 990) uses `ConvertStroopValueToReal()` while `reserve_a_deposit_amount` (line 991-995) uses `ParseFloat(amount.String())`. Both produce float64 but via different paths.

5. **Not a duplicate**: Published finding `utilities/001-float64-stroop-collisions` covers `ConvertStroopValueToReal` callers (account, trustline, offer, trade). Published finding `data-integrity/002-token-transfer-float64-rounding` covers `transformEvents` in token_transfer.go. This hypothesis targets a distinct code path (`extractOperationDetails` lines 991-1019) using a distinct mechanism (`ParseFloat(amount.String())`) affecting distinct output fields.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go` (append to existing LP deposit tests around line 1751)
- **Setup**: Construct an `ingest.LedgerTransaction` with a successful `LiquidityPoolDeposit` operation where the post-deposit reserve delta is `9007199254740993` stroops (2^53 + 1). Use `getLiquidityPoolAndProductDelta`-compatible ledger changes with pre/post `LiquidityPoolEntry` values differing by that delta.
- **Steps**: Call `extractOperationDetails()` (or `TransformOperation()`) with the constructed transaction and operation.
- **Assertion**: Assert that `details["reserve_a_deposit_amount"]` equals the exact float64 representation of `900719925.4740993`. The test should fail because the actual value will be `900719925.4740992` (or its float64 neighbor), proving the one-stroop precision loss. Alternatively, construct two transactions differing by 1 stroop in the deposit delta and assert their exported `reserve_a_deposit_amount` values differ.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestLPDepositFloat64PrecisionLoss"
**Test Language**: Go

### Demonstration

The test constructs two `LiquidityPoolDeposit` transactions with reserve deltas differing by exactly 1 stroop (90071992547409930 vs 90071992547409931). The production code path `extractOperationDetails` → `strconv.ParseFloat(amount.String(...), 64)` collapses both to the identical float64 value `9.007199254740993e+09`, which formats as `"9007199254.7409935"` instead of the exact `"9007199254.7409930"` or `"9007199254.7409931"`. This proves three financial fields (`reserve_a_deposit_amount`, `reserve_b_deposit_amount`, `shares_received`) silently corrupt large deposit values. Note: the original hypothesis used 2^53+1 (9007199254740993 stroops), which produces a 16-significant-digit decimal that float64 can still round-trip; the actual trigger requires ~17 significant digits (e.g., 9×10^16 stroops), which is well within `xdr.Int64` range.

### Test Body

```go
package transform

import (
	"strconv"
	"testing"

	"github.com/stellar/go-stellar-sdk/amount"
	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestLPDepositFloat64PrecisionLoss demonstrates that liquidity_pool_deposit
// detail fields lose precision when large xdr.Int64 reserve deltas are
// converted through amount.String() → strconv.ParseFloat(). Two deposits
// differing by 1 stroop produce identical exported values due to float64
// rounding in the ParseFloat step.
func TestLPDepositFloat64PrecisionLoss(t *testing.T) {
	// At this scale, the 7-decimal representation has 17 significant digits,
	// exceeding float64's ~15.95 digit precision. Two adjacent stroop values
	// produce distinct exact strings but collapse to the same float64.
	const (
		stroopA int64 = 90071992547409930 // produces "9007199254.7409930"
		stroopB int64 = 90071992547409931 // produces "9007199254.7409931"
	)

	poolID := xdr.PoolId{1, 2, 3, 4, 5, 6, 7, 8, 9}

	buildTx := func(depositDelta xdr.Int64) (xdr.Operation, ingest.LedgerTransaction) {
		sourceAccount, _ := xdr.NewMuxedAccount(
			xdr.CryptoKeyTypeKeyTypeEd25519, xdr.Uint256([32]byte{}))

		op := xdr.Operation{
			SourceAccount: &sourceAccount,
			Body: xdr.OperationBody{
				Type: xdr.OperationTypeLiquidityPoolDeposit,
				LiquidityPoolDepositOp: &xdr.LiquidityPoolDepositOp{
					LiquidityPoolId: poolID,
					MaxAmountA:      xdr.Int64(depositDelta + 1000),
					MaxAmountB:      100,
					MinPrice:        xdr.Price{N: 1, D: 1},
					MaxPrice:        xdr.Price{N: 1, D: 1},
				},
			},
		}

		// Operation result for a successful LP deposit
		opResults := []xdr.OperationResult{
			{
				Code: xdr.OperationResultCodeOpInner,
				Tr: &xdr.OperationResultTr{
					Type: xdr.OperationTypeLiquidityPoolDeposit,
					LiquidityPoolDepositResult: &xdr.LiquidityPoolDepositResult{
						Code: xdr.LiquidityPoolDepositResultCodeLiquidityPoolDepositSuccess,
					},
				},
			},
		}

		assetA := xdr.Asset{Type: xdr.AssetTypeAssetTypeNative}
		assetB := xdr.Asset{
			Type: xdr.AssetTypeAssetTypeCreditAlphanum4,
			AlphaNum4: &xdr.AlphaNum4{
				AssetCode: xdr.AssetCode4([4]byte{0x55, 0x53, 0x53, 0x44}),
				Issuer:    testAccount4ID,
			},
		}

		tx := ingest.LedgerTransaction{
			Index: 1,
			Envelope: xdr.TransactionEnvelope{
				Type: xdr.EnvelopeTypeEnvelopeTypeTx,
				V1: &xdr.TransactionV1Envelope{
					Tx: xdr.Transaction{
						SourceAccount: sourceAccount,
						Operations:    []xdr.Operation{op},
					},
				},
			},
			Result: xdr.TransactionResultPair{
				Result: xdr.TransactionResult{
					Result: xdr.TransactionResultResult{
						Code:    xdr.TransactionResultCodeTxSuccess,
						Results: &opResults,
					},
				},
			},
			UnsafeMeta: xdr.TransactionMeta{
				V: 1,
				V1: &xdr.TransactionMetaV1{
					Operations: []xdr.OperationMeta{
						{
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
															AssetA: assetA,
															AssetB: assetB,
															Fee:    30,
														},
														ReserveA:                 0,
														ReserveB:                 0,
														TotalPoolShares:          0,
														PoolSharesTrustLineCount: 1,
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
															AssetA: assetA,
															AssetB: assetB,
															Fee:    30,
														},
														ReserveA:                 depositDelta,
														ReserveB:                 100,
														TotalPoolShares:          200,
														PoolSharesTrustLineCount: 2,
													},
												},
											},
										},
									},
								},
							},
						},
					},
				},
			},
		}
		return op, tx
	}

	// --- Sub-test 1: Verify the exact string is available but lost ---
	t.Run("exact_string_lost_in_ParseFloat", func(t *testing.T) {
		exactStr := amount.String(xdr.Int64(stroopB))
		if exactStr != "9007199254.7409931" {
			t.Fatalf("amount.String produced unexpected result: %s", exactStr)
		}

		// Parse to float64 (the production code path) and format back
		parsed, err := strconv.ParseFloat(exactStr, 64)
		if err != nil {
			t.Fatal(err)
		}
		roundTripped := strconv.FormatFloat(parsed, 'f', 7, 64)

		if roundTripped == exactStr {
			t.Fatal("Expected float64 round-trip to lose precision, but it didn't")
		}
		t.Logf("Precision loss confirmed in ParseFloat: %q → float64 → %q", exactStr, roundTripped)
	})

	// --- Sub-test 2: Two different inputs produce identical output ---
	t.Run("one_stroop_difference_collapsed", func(t *testing.T) {
		opA, txA := buildTx(xdr.Int64(stroopA))
		opB, txB := buildTx(xdr.Int64(stroopB))

		detailsA, err := extractOperationDetails(opA, txA, 0, "")
		if err != nil {
			t.Fatalf("extractOperationDetails for stroopA: %v", err)
		}
		detailsB, err := extractOperationDetails(opB, txB, 0, "")
		if err != nil {
			t.Fatalf("extractOperationDetails for stroopB: %v", err)
		}

		amtA := detailsA["reserve_a_deposit_amount"].(float64)
		amtB := detailsB["reserve_a_deposit_amount"].(float64)

		// These are two deposits differing by 1 stroop. A correct export
		// would produce distinct values. The bug causes them to collide.
		if amtA == amtB {
			t.Logf("BUG CONFIRMED: two deposits differing by 1 stroop produce identical output")
			t.Logf("  deposit %d stroops → %v", stroopA, amtA)
			t.Logf("  deposit %d stroops → %v", stroopB, amtB)
		} else {
			t.Fatal("Expected the two values to collide due to float64 precision, but they differ")
		}
	})

	// --- Sub-test 3: Verify via TransformOperation (full production path) ---
	t.Run("full_transform_precision_loss", func(t *testing.T) {
		_, tx := buildTx(xdr.Int64(stroopB))

		lcm := xdr.LedgerCloseMeta{
			V: 0,
			V0: &xdr.LedgerCloseMetaV0{
				LedgerHeader: xdr.LedgerHeaderHistoryEntry{
					Header: xdr.LedgerHeader{
						LedgerSeq: 1,
					},
				},
			},
		}

		output, err := TransformOperation(tx.Envelope.V1.Tx.Operations[0], 0, tx, 1, lcm, "")
		if err != nil {
			t.Fatalf("TransformOperation: %v", err)
		}

		got := output.OperationDetails["reserve_a_deposit_amount"].(float64)
		gotStr := strconv.FormatFloat(got, 'f', 7, 64)
		wantStr := "9007199254.7409931"

		if gotStr == wantStr {
			t.Fatal("Expected precision loss in TransformOperation output, but value was exact")
		}
		t.Logf("BUG CONFIRMED via TransformOperation: got %s, want %s", gotStr, wantStr)
	})
}
```

### Test Output

```
=== RUN   TestLPDepositFloat64PrecisionLoss
=== RUN   TestLPDepositFloat64PrecisionLoss/exact_string_lost_in_ParseFloat
    data_integrity_poc_test.go:169: Precision loss confirmed in ParseFloat: "9007199254.7409931" → float64 → "9007199254.7409935"
=== RUN   TestLPDepositFloat64PrecisionLoss/one_stroop_difference_collapsed
    data_integrity_poc_test.go:192: BUG CONFIRMED: two deposits differing by 1 stroop produce identical output
    data_integrity_poc_test.go:193:   deposit 90071992547409930 stroops → 9.007199254740993e+09
    data_integrity_poc_test.go:194:   deposit 90071992547409931 stroops → 9.007199254740993e+09
=== RUN   TestLPDepositFloat64PrecisionLoss/full_transform_precision_loss
    data_integrity_poc_test.go:227: BUG CONFIRMED via TransformOperation: got 9007199254.7409935, want 9007199254.7409931
--- PASS: TestLPDepositFloat64PrecisionLoss (0.00s)
    --- PASS: TestLPDepositFloat64PrecisionLoss/exact_string_lost_in_ParseFloat (0.00s)
    --- PASS: TestLPDepositFloat64PrecisionLoss/one_stroop_difference_collapsed (0.00s)
    --- PASS: TestLPDepositFloat64PrecisionLoss/full_transform_precision_loss (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.809s
```
