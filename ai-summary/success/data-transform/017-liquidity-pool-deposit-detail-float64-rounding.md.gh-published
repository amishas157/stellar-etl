# 017: Liquidity-pool deposit details round large reserve deltas and share amounts

**Date**: 2026-04-11
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformOperation()` exports successful `liquidity_pool_deposit` reserve deltas and share deltas as `float64` after first computing the exact 7-decimal string with `amount.String()`. For sufficiently large deposits, two on-chain values that differ by 1 stroop collapse to the same exported JSON number, so `reserve_a_deposit_amount`, `reserve_b_deposit_amount`, and `shares_received` become silently wrong while still looking plausible.

## Root Cause

In the `LiquidityPoolDeposit` branch of `extractOperationDetails()`, the transform gets exact `xdr.Int64` deltas from `getLiquidityPoolAndProductDelta()`, converts them to exact decimal strings with `amount.String()`, and then immediately parses those strings back into `float64` with `strconv.ParseFloat(..., 64)`. That last conversion discards low-order digits once the scaled decimal exceeds float64 precision, even though the exact string representation was already available.

## Reproduction

Construct a successful `liquidity_pool_deposit` whose reserve and share deltas are large enough that their 7-decimal form needs more than float64's exact precision, such as `90071992547409931` stroops (`9007199254.7409931`). Run the normal `TransformOperation()` path and observe that the exported detail fields round to a neighboring value, and that two transactions differing by 1 stroop export the same numbers.

## Affected Code

- `internal/transform/operation.go:extractOperationDetails:957-1019` — converts exact liquidity-pool deposit deltas to strings and then back to lossy `float64` values
- `internal/transform/operation.go:getLiquidityPoolAndProductDelta:238-285` — returns the exact `xdr.Int64` reserve and share deltas consumed by the buggy branch
- `internal/transform/operation.go:transactionOperationWrapper.Details:1603-1649` — sibling LP-deposit formatter keeps the exact `amount.String()` values, showing precise output was available

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestLPDepositFloat64PrecisionLoss`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

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

func TestLPDepositFloat64PrecisionLoss(t *testing.T) {
	const (
		stroopA int64 = 90071992547409930
		stroopB int64 = 90071992547409931
	)

	poolID := xdr.PoolId{1, 2, 3, 4, 5, 6, 7, 8, 9}

	buildTx := func(depositDelta xdr.Int64) (xdr.Operation, ingest.LedgerTransaction) {
		sourceAccount, _ := xdr.NewMuxedAccount(
			xdr.CryptoKeyTypeKeyTypeEd25519,
			xdr.Uint256([32]byte{}),
		)

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
														ReserveB:                 depositDelta,
														TotalPoolShares:          depositDelta,
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

	t.Run("parsefloat_loses_exact_decimal", func(t *testing.T) {
		exact := amount.String(xdr.Int64(stroopB))
		if exact != "9007199254.7409931" {
			t.Fatalf("unexpected exact string: %s", exact)
		}

		parsed, err := strconv.ParseFloat(exact, 64)
		if err != nil {
			t.Fatal(err)
		}

		if got := strconv.FormatFloat(parsed, 'f', 7, 64); got == exact {
			t.Fatalf("expected precision loss, got exact round-trip %s", got)
		}
	})

	t.Run("extract_operation_details_collapses_adjacent_values", func(t *testing.T) {
		opA, txA := buildTx(xdr.Int64(stroopA))
		opB, txB := buildTx(xdr.Int64(stroopB))

		detailsA, err := extractOperationDetails(opA, txA, 0, "")
		if err != nil {
			t.Fatalf("extractOperationDetails A: %v", err)
		}
		detailsB, err := extractOperationDetails(opB, txB, 0, "")
		if err != nil {
			t.Fatalf("extractOperationDetails B: %v", err)
		}

		for _, key := range []string{
			"reserve_a_deposit_amount",
			"reserve_b_deposit_amount",
			"shares_received",
		} {
			gotA := detailsA[key].(float64)
			gotB := detailsB[key].(float64)
			if gotA != gotB {
				t.Fatalf("%s stayed distinct: %v vs %v", key, gotA, gotB)
			}
		}
	})

	t.Run("transform_operation_exports_rounded_value", func(t *testing.T) {
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
		if gotStr := strconv.FormatFloat(got, 'f', 7, 64); gotStr == "9007199254.7409931" {
			t.Fatalf("expected rounded output, got exact value %s", gotStr)
		}
	})
}
```

## Expected vs Actual Behavior

- **Expected**: successful liquidity-pool deposit exports preserve the exact 7-decimal reserve deltas and share delta implied by ledger changes, so a 1-stroop difference remains visible in JSON output
- **Actual**: `reserve_a_deposit_amount`, `reserve_b_deposit_amount`, and `shares_received` round through `float64`, so adjacent large values collapse to the same exported number

## Adversarial Review

1. Exercises claimed bug: YES — the test constructs real ledger changes, calls both `extractOperationDetails()` and `TransformOperation()`, and observes the wrong exported detail values on the production path.
2. Realistic preconditions: YES — the trigger only requires large but valid `xdr.Int64` reserve/share deltas; `90071992547409931` stroops is about 9.0 billion XLM, below Stellar's total supply and also achievable for issued assets.
3. Bug vs by-design: BUG — the code already has the exact decimal string from `amount.String()` and sibling LP formatters export that exact string, so the lossy `ParseFloat` round-trip is unnecessary and inconsistent.
4. Final severity: Critical — these are monetary fields in exported operation rows, and the transform silently publishes wrong numeric values.
5. In scope: YES — this is a concrete transform-layer data corruption bug in normal export output, not an upstream SDK issue or a test-only artifact.
6. Test correctness: CORRECT — the assertions do not restate setup values; they prove that two distinct on-chain deltas collapse to one exported number and that the full transform path emits the rounded value.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Stop converting these fields back into `float64`. Preserve the exact `amount.String()` output for `reserve_a_deposit_amount`, `reserve_b_deposit_amount`, and `shares_received`, matching the sibling LP formatter and effect output paths. If numeric types are required elsewhere, use an exact decimal representation rather than `float64`.
