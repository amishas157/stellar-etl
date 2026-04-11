# 019: Transfer-style operation details round large amounts

**Date**: 2026-04-11
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformOperation()` exports several transfer-style operation detail amounts as `float64` in `history_operations.details`, so distinct large stroop amounts can collapse to the same JSON number. For `create_account`, `payment`, `create_claimable_balance`, and `clawback`, two valid on-chain values that differ by 1 stroop can become indistinguishable even though the same package already has exact decimal-string formatting for these fields.

## Root Cause

`extractOperationDetails()` sends these amount fields through `utils.ConvertStroopValueToReal()`, which returns the nearest `float64` to the decimal value. Once the scaled amount exceeds float64 precision, low-order stroop digits are rounded away. That loss is avoidable here because `OperationOutput.OperationDetails` is a `map[string]interface{}` and sibling code in `transactionOperationWrapper.Details()` already uses exact `amount.String(...)` values for the same logical fields.

## Reproduction

Create two otherwise identical successful `create_account` operations whose `StartingBalance` values differ by 1 stroop: `90071992547409930` and `90071992547409931`. Run both through `TransformOperation()` and compare `OperationDetails["starting_balance"]`: both export as the same `float64` value even though `amount.String(...)` preserves the exact decimal strings `9007199254.7409930` and `9007199254.7409931`.

## Affected Code

- `internal/transform/operation.go:590-614` — `create_account` and `payment` detail amounts are converted with `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:881-885` — `create_claimable_balance.amount` uses the same lossy conversion
- `internal/transform/operation.go:924-932` — `clawback.amount` uses the same lossy conversion
- `internal/transform/operation.go:1368-1378` — sibling formatter keeps exact `amount.String(...)` values for `create_account` and `payment`
- `internal/transform/operation.go:1537-1541` — sibling formatter keeps exact `amount.String(...)` for `create_claimable_balance`
- `internal/transform/operation.go:1579-1583` — sibling formatter keeps exact `amount.String(...)` for `clawback`
- `internal/transform/schema.go:136-150` — operation details are exported through `map[string]interface{}`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestPoCCreateAccountStartingBalancePrecisionLoss`
- **Test language**: `go`
- **How to run**: Create the target test file with the body below, then run `go test ./internal/transform/... -run TestPoCCreateAccountStartingBalancePrecisionLoss -v`.

### Test Body

```go
package transform

import (
	"fmt"
	"testing"

	"github.com/stellar/go-stellar-sdk/amount"
	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
	"github.com/stellar/stellar-etl/v2/internal/utils"
	"github.com/stretchr/testify/assert"
)

func TestPoCCreateAccountStartingBalancePrecisionLoss(t *testing.T) {
	tt := assert.New(t)

	stroopValA := xdr.Int64(90071992547409930)
	stroopValB := xdr.Int64(90071992547409931)

	exactA := amount.String(stroopValA)
	exactB := amount.String(stroopValB)
	tt.NotEqual(exactA, exactB, "exact string representations should differ")

	destAccount := xdr.MustAddress("GAUJETIZVEP2NRYLUESJ3LS66NVCEGMON4UDCBCSBEVPIID773P2W6AY")
	makeCreateAccountOp := func(balance xdr.Int64) xdr.Operation {
		return xdr.Operation{
			SourceAccount: &genericSourceAccount,
			Body: xdr.OperationBody{
				Type: xdr.OperationTypeCreateAccount,
				CreateAccountOp: &xdr.CreateAccountOp{
					Destination:     destAccount,
					StartingBalance: balance,
				},
			},
		}
	}

	opA := makeCreateAccountOp(stroopValA)
	opB := makeCreateAccountOp(stroopValB)

	makeEnvelope := func(op xdr.Operation) xdr.TransactionV1Envelope {
		return xdr.TransactionV1Envelope{
			Tx: xdr.Transaction{
				SourceAccount: genericSourceAccount,
				Memo:          xdr.Memo{},
				Operations:    []xdr.Operation{op},
				Ext: xdr.TransactionExt{
					V: 0,
					SorobanData: &xdr.SorobanTransactionData{
						Ext:       xdr.SorobanTransactionDataExt{V: 0},
						Resources: xdr.SorobanResources{Footprint: xdr.LedgerFootprint{}},
					},
				},
			},
		}
	}

	envA := makeEnvelope(opA)
	envB := makeEnvelope(opB)

	makeTx := func(env *xdr.TransactionV1Envelope) ingest.LedgerTransaction {
		return ingest.LedgerTransaction{
			Index: 1,
			Envelope: xdr.TransactionEnvelope{
				Type: xdr.EnvelopeTypeEnvelopeTypeTx,
				V1:   env,
			},
			Result:     utils.CreateSampleResultMeta(true, 1).Result,
			UnsafeMeta: xdr.TransactionMeta{V: 1, V1: genericTxMeta},
		}
	}

	txA := makeTx(&envA)
	txB := makeTx(&envB)

	ledgerCloseMeta := xdr.LedgerCloseMeta{
		V: 0,
		V0: &xdr.LedgerCloseMetaV0{
			LedgerHeader: xdr.LedgerHeaderHistoryEntry{},
		},
	}

	outputA, err := TransformOperation(opA, 0, txA, 1, ledgerCloseMeta, "testnet")
	tt.NoError(err)

	outputB, err := TransformOperation(opB, 0, txB, 1, ledgerCloseMeta, "testnet")
	tt.NoError(err)

	balanceA := outputA.OperationDetails["starting_balance"]
	balanceB := outputB.OperationDetails["starting_balance"]

	floatA, okA := balanceA.(float64)
	floatB, okB := balanceB.(float64)
	tt.True(okA)
	tt.True(okB)
	tt.Equal(floatA, floatB, "two different stroop values collapse to the same exported float64")

	fmt.Printf("exact A: %s\n", exactA)
	fmt.Printf("exact B: %s\n", exactB)
	fmt.Printf("exported A: %.10f\n", floatA)
	fmt.Printf("exported B: %.10f\n", floatB)
}
```

## Expected vs Actual Behavior

- **Expected**: transfer-style operation details should preserve distinct stroop values, either by exporting an exact decimal string or by otherwise keeping adjacent 7-decimal amounts distinguishable.
- **Actual**: large values round to the same `float64`, so different on-chain amounts export as the same plausible JSON number.

## Adversarial Review

1. Exercises claimed bug: YES — the test builds real `xdr.Operation` values, runs the production `TransformOperation()` path, and compares the exported `starting_balance` detail values.
2. Realistic preconditions: YES — the affected Stellar operation amount fields are `xdr.Int64`, and the trigger value is representable by normal on-chain operations.
3. Bug vs by-design: BUG — mixed numeric and string values are already present in operation-detail JSON, and the same file already contains exact string formatting for these amount fields, so this precision loss is a conversion choice rather than a schema requirement.
4. Final severity: Critical — this silently corrupts exported monetary values in a production output surface.
5. In scope: YES — it is a concrete data-correctness issue in `internal/transform/`.
6. Test correctness: CORRECT — the assertion checks a real mismatch between exact decimal output and the value actually exported by the production code.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Stop exporting these transfer-style detail amounts as `float64`. Use exact decimal strings such as `amount.String(...)` for `starting_balance` and the affected `amount` fields, consistent with the sibling formatter already present in `operation.go`.
