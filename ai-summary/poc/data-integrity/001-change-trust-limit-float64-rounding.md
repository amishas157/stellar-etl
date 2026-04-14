# H001: `change_trust` limit rounds distinct large stroop values together

**Date**: 2026-04-14
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

A successful `change_trust` operation should export the exact trustline limit implied by the on-chain `ChangeTrustOp.Limit` stroop value. Two valid limits that differ by 1 stroop, such as `90071992547409930` and `90071992547409931`, should remain distinguishable in `history_operations.details.limit`.

## Mechanism

The live `extractOperationDetails()` path converts `ChangeTrustOp.Limit` through `utils.ConvertStroopValueToReal()`, which returns the nearest `float64` to the 7-decimal stroop value. At sufficiently large magnitudes, adjacent limits collapse to the same JSON number even though the package's sibling formatter already demonstrates that this field can be rendered exactly with `amount.String(op.Limit)`.

## Trigger

Run `TransformOperation()` on two otherwise identical successful `change_trust` operations whose `Limit` values are `90071992547409930` and `90071992547409931` stroops. Inspect `details["limit"]`: both rows should export the same `float64` even though the exact decimal strings differ by one stroop.

## Target Code

- `internal/transform/operation.go:817-820` — live `change_trust` branch writes `details["limit"]` via `ConvertStroopValueToReal`
- `internal/transform/operation.go:1495-1506` — sibling formatter keeps the same field exact with `amount.String(op.Limit)`
- `internal/utils/main.go:84-87` — `ConvertStroopValueToReal()` returns a `float64`
- `.../xdr/xdr_generated.go:27022-27024` — `ChangeTrustOp.Limit` is exact XDR `Int64`

## Evidence

`ChangeTrustOp.Limit` is an exact `Int64`, but the production exporter stores it in a `map[string]interface{}` as a `float64` anyway. The duplicate formatter in the same file uses `amount.String(op.Limit)`, showing there is no schema constraint forcing the live path to round away the low-order stroops.

## Anti-Evidence

The corruption only appears once the scaled value exceeds float64's exact 7-decimal precision; ordinary trustline limits still serialize distinctly. The field has no separate raw-stroop companion column today, so the rounded JSON value is the only value downstream sees.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-14
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the `change_trust` branch in `extractOperationDetails()` (line 820) which stores `details["limit"] = utils.ConvertStroopValueToReal(op.Limit)`. The `ConvertStroopValueToReal()` function (utils/main.go:85-87) uses `big.NewRat(int64(input), 10000000).Float64()`, which returns the nearest `float64`. For stroop values whose scaled result exceeds `float64`'s 53-bit mantissa precision (~15.95 decimal digits), adjacent stroop values collapse to the same `float64`. The sibling formatter at line 1506 in `transactionOperationWrapper.Details()` uses `amount.String(op.Limit)` which preserves full precision, confirming the exact representation is available but unused in the production export path.

### Code Paths Examined

- `internal/transform/operation.go:30-100` — `TransformOperation()` calls `extractOperationDetails()` and stores result in `OperationOutput.OperationDetails`
- `internal/transform/operation.go:584-820` — `extractOperationDetails()` `change_trust` case stores `details["limit"]` as `float64` via `ConvertStroopValueToReal`
- `internal/utils/main.go:84-87` — `ConvertStroopValueToReal()` returns `big.NewRat(int64(input), 10000000).Float64()` — confirmed float64 output
- `internal/transform/operation.go:1364-1506` — `transactionOperationWrapper.Details()` uses `amount.String(op.Limit)` for the same field — exact string representation
- `internal/transform/schema.go:142,149` — `OperationOutput.OperationDetails` and `OperationDetailsJSON` are both `map[string]interface{}`, so the `float64` value flows directly to JSON serialization

### Findings

1. **Confirmed precision loss**: At the magnitude of `9007199254.74` (scaled XLM), the float64 ULP is `2^(33-52) = 2^-19 ≈ 1.907e-6`. A 1-stroop difference maps to `1e-7` in scaled value, which is ~19x smaller than the ULP. Adjacent stroop values collide starting around `4.5 × 10^15` stroops (~450 million XLM scaled).

2. **Default trustline limit is affected**: The default/maximum trustline limit `Int64.MaxValue = 9223372036854775807` is well above the collision threshold, so the default limit value is also lossy in exports.

3. **Widespread pattern**: `ConvertStroopValueToReal()` is used for 30+ financial fields across operation details (amounts, limits, reserves), account balances, trustline balances, offer amounts, trade amounts, and liquidity pool reserves. The `change_trust` limit is one concrete instance of a systemic pattern.

4. **Not a duplicate of success/002**: Success finding 002 covers token transfers using `strconv.ParseFloat` in `token_transfer.go`. This hypothesis targets operation details using `ConvertStroopValueToReal` in `operation.go` — different code path, different export table.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go` (or create `internal/transform/data_integrity_poc_test.go`)
- **Setup**: Create two `ChangeTrustOp` operations with `Limit` values of `90071992547409930` and `90071992547409931` (XDR Int64). Wrap each in a minimal `xdr.Operation` and `ingest.LedgerTransaction` using the existing `utils.CreateSampleTx()` test helpers.
- **Steps**: Call `TransformOperation()` for each operation. Extract `details["limit"]` from both `OperationOutput.OperationDetails` maps.
- **Assertion**: Assert that the two `details["limit"]` values are NOT equal (this assertion will FAIL, demonstrating the bug). Alternatively, assert they ARE equal to demonstrate the collision — both float64 values should be identical despite distinct stroop inputs. Also verify that `amount.String()` on the same two Int64 values produces distinct strings, confirming the exact representation is available.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-14
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestChangeTrustLimitFloat64Rounding"
**Test Language**: Go

### Demonstration

The test constructs two `change_trust` operations with `Limit` values differing by 1 stroop (90071992547409930 vs 90071992547409931), runs both through `TransformOperation()`, and confirms that the exported `details["limit"]` float64 values are identical despite distinct inputs. Meanwhile, `amount.String()` correctly distinguishes the two values ("9007199254.7409930" vs "9007199254.7409931"), proving the exact representation is available but unused in the production export path.

### Test Body

```go
func TestChangeTrustLimitFloat64Rounding(t *testing.T) {
	// Two adjacent stroop values that exceed float64's 53-bit mantissa precision
	// when scaled to XLM (divided by 1e7). These should be distinguishable but
	// ConvertStroopValueToReal returns the same float64 for both.
	limitA := xdr.Int64(90071992547409930)
	limitB := xdr.Int64(90071992547409931)

	// Verify the exact string representation distinguishes them
	strA := amount.String(limitA)
	strB := amount.String(limitB)
	if strA == strB {
		t.Fatalf("amount.String should distinguish the two limits, but both are %s", strA)
	}
	t.Logf("amount.String(limitA) = %s", strA)
	t.Logf("amount.String(limitB) = %s", strB)

	// Build two change_trust operations with the two different limits
	opA := xdr.Operation{
		SourceAccount: &genericSourceAccount,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeChangeTrust,
			ChangeTrustOp: &xdr.ChangeTrustOp{
				Line:  usdtChangeTrustAsset,
				Limit: limitA,
			},
		},
	}
	opB := xdr.Operation{
		SourceAccount: &genericSourceAccount,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeChangeTrust,
			ChangeTrustOp: &xdr.ChangeTrustOp{
				Line:  usdtChangeTrustAsset,
				Limit: limitB,
			},
		},
	}

	// Build transaction envelopes and results
	changeTrustResult := xdr.OperationResult{
		Code: xdr.OperationResultCodeOpInner,
		Tr: &xdr.OperationResultTr{
			Type: xdr.OperationTypeChangeTrust,
			ChangeTrustResult: &xdr.ChangeTrustResult{
				Code: xdr.ChangeTrustResultCodeChangeTrustSuccess,
			},
		},
	}
	opResults := []xdr.OperationResult{changeTrustResult}

	envelopeA := xdr.TransactionV1Envelope{
		Tx: xdr.Transaction{
			SourceAccount: genericSourceAccount,
			Memo:          xdr.Memo{},
			Operations:    []xdr.Operation{opA},
			Ext: xdr.TransactionExt{
				V: 0,
				SorobanData: &xdr.SorobanTransactionData{
					Ext:         xdr.SorobanTransactionDataExt{V: 0},
					Resources:   xdr.SorobanResources{Footprint: xdr.LedgerFootprint{}},
					ResourceFee: 0,
				},
			},
		},
	}
	txA := ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1:   &envelopeA,
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
			V:  1,
			V1: utils.CreateSampleTxMeta(1, lpAssetA, lpAssetB),
		},
	}

	envelopeB := xdr.TransactionV1Envelope{
		Tx: xdr.Transaction{
			SourceAccount: genericSourceAccount,
			Memo:          xdr.Memo{},
			Operations:    []xdr.Operation{opB},
			Ext: xdr.TransactionExt{
				V: 0,
				SorobanData: &xdr.SorobanTransactionData{
					Ext:         xdr.SorobanTransactionDataExt{V: 0},
					Resources:   xdr.SorobanResources{Footprint: xdr.LedgerFootprint{}},
					ResourceFee: 0,
				},
			},
		},
	}
	txB := ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1:   &envelopeB,
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
			V:  1,
			V1: utils.CreateSampleTxMeta(1, lpAssetA, lpAssetB),
		},
	}

	lcm := makeLedgerCloseMeta()

	outputA, err := TransformOperation(opA, 0, txA, 1, lcm, "")
	if err != nil {
		t.Fatalf("TransformOperation for limitA failed: %v", err)
	}
	outputB, err := TransformOperation(opB, 0, txB, 1, lcm, "")
	if err != nil {
		t.Fatalf("TransformOperation for limitB failed: %v", err)
	}

	exportedLimitA := outputA.OperationDetails["limit"]
	exportedLimitB := outputB.OperationDetails["limit"]

	t.Logf("Input limitA (stroops): %d", limitA)
	t.Logf("Input limitB (stroops): %d", limitB)
	t.Logf("Exported limitA: %v (type %T)", exportedLimitA, exportedLimitA)
	t.Logf("Exported limitB: %v (type %T)", exportedLimitB, exportedLimitB)

	// The bug: two distinct stroop values produce the same float64 output.
	// This assertion PASSES when the bug exists — both limits are identical.
	floatA, okA := exportedLimitA.(float64)
	floatB, okB := exportedLimitB.(float64)
	if !okA || !okB {
		t.Fatalf("Expected float64 limits, got %T and %T", exportedLimitA, exportedLimitB)
	}

	if floatA == floatB {
		t.Errorf("BUG CONFIRMED: Two distinct stroop values (diff=1) exported as identical float64.\n"+
			"  limitA stroops: %d\n"+
			"  limitB stroops: %d\n"+
			"  exported float64 (both): %.20f\n"+
			"  exact string A: %s\n"+
			"  exact string B: %s\n"+
			"Downstream consumers cannot distinguish these trustline limits.",
			limitA, limitB, floatA, strA, strB)
	}
}
```

### Test Output

```
=== RUN   TestChangeTrustLimitFloat64Rounding
    data_integrity_poc_test.go:316: amount.String(limitA) = 9007199254.7409930
    data_integrity_poc_test.go:317: amount.String(limitB) = 9007199254.7409931
    data_integrity_poc_test.go:437: Input limitA (stroops): 90071992547409930
    data_integrity_poc_test.go:438: Input limitB (stroops): 90071992547409931
    data_integrity_poc_test.go:439: Exported limitA: 9.007199254740993e+09 (type float64)
    data_integrity_poc_test.go:440: Exported limitB: 9.007199254740993e+09 (type float64)
    data_integrity_poc_test.go:451: BUG CONFIRMED: Two distinct stroop values (diff=1) exported as identical float64.
          limitA stroops: 90071992547409930
          limitB stroops: 90071992547409931
          exported float64 (both): 9007199254.74099349975585937500
          exact string A: 9007199254.7409930
          exact string B: 9007199254.7409931
        Downstream consumers cannot distinguish these trustline limits.
--- FAIL: TestChangeTrustLimitFloat64Rounding (0.00s)
FAIL
```
