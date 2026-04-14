# 024: change_trust limit float64 rounding exports the wrong amount

**Date**: 2026-04-14
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

Successful `change_trust` operations export `details.limit` by converting the exact XDR `Int64` limit into a `float64`. Once the scaled asset amount is large enough, adjacent one-stroop limits collapse to the same exported JSON number even though Stellar already provides an exact decimal formatter for this field.

This happens in the normal `TransformOperation()` path, not just in test helpers. For limits `90071992547409930` and `90071992547409931`, the exporter produces the same serialized `details` JSON with `limit: 9007199254.740993`, so downstream consumers cannot distinguish two different on-chain trustline caps.

## Root Cause

`extractOperationDetails()` handles `change_trust` by storing `details["limit"] = utils.ConvertStroopValueToReal(op.Limit)`. `ConvertStroopValueToReal()` converts the exact stroop integer through `big.Rat.Float64()`, so the value is rounded to the nearest IEEE-754 `float64` before it ever reaches JSON or Parquet serialization.

The same file already contains a sibling formatter that uses `amount.String(op.Limit)` for `change_trust`, and the trustline export keeps the raw limit as `int64`, showing exact export is feasible. The corruption is therefore caused by this specific lossy conversion, not by an unavoidable schema constraint.

## Reproduction

Any normal export that includes a successful `change_trust` operation with a sufficiently large limit can hit this path. The collision threshold is far below Stellar's `Int64` maximum, and this repository's own trustline fixtures already use multi-quadrillion-stroop limits, so the precondition is realistic for issued-asset trustlines.

When two adjacent limits above that threshold are transformed, `TransformOperation()` returns identical `float64` values in `OperationDetails["limit"]`, and `json.Marshal` serializes both rows with the same `details.limit` number.

## Affected Code

- `internal/transform/operation.go:TransformOperation:29-100` — copies the lossy `extractOperationDetails()` map directly into `details` and `details_json`
- `internal/transform/operation.go:extractOperationDetails:800-820` — `change_trust` stores `limit` via `utils.ConvertStroopValueToReal(op.Limit)`
- `internal/utils/main.go:ConvertStroopValueToReal:84-87` — converts exact stroop integers to nearest `float64`
- `internal/transform/parquet_converter.go:toJSONString:19-24` and `internal/transform/parquet_converter.go:OperationOutput.ToParquet:147-160` — serialize the same lossy map for Parquet output

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestChangeTrustLimitFloat64Rounding`
- **Test language**: `go`
- **How to run**: `cd <repo-root> && go build ./... && go test ./internal/transform/... -run TestChangeTrustLimitFloat64Rounding -v` after creating the target test file with the body below.

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"

	"github.com/stellar/go-stellar-sdk/amount"
	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestChangeTrustLimitFloat64Rounding(t *testing.T) {
	limitA := xdr.Int64(90071992547409930)
	limitB := xdr.Int64(90071992547409931)

	exactA := amount.String(limitA)
	exactB := amount.String(limitB)
	if exactA == exactB {
		t.Fatalf("expected exact formatter to distinguish the limits, got %q", exactA)
	}

	changeTrustSuccess := []xdr.OperationResult{
		{
			Code: xdr.OperationResultCodeOpInner,
			Tr: &xdr.OperationResultTr{
				Type: xdr.OperationTypeChangeTrust,
				ChangeTrustResult: &xdr.ChangeTrustResult{
					Code: xdr.ChangeTrustResultCodeChangeTrustSuccess,
				},
			},
		},
	}

	buildTransaction := func(limit xdr.Int64) (xdr.Operation, ingest.LedgerTransaction) {
		op := xdr.Operation{
			Body: xdr.OperationBody{
				Type: xdr.OperationTypeChangeTrust,
				ChangeTrustOp: &xdr.ChangeTrustOp{
					Line:  usdtChangeTrustAsset,
					Limit: limit,
				},
			},
		}

		tx := ingest.LedgerTransaction{
			Index: 1,
			Envelope: xdr.TransactionEnvelope{
				Type: xdr.EnvelopeTypeEnvelopeTypeTx,
				V1: &xdr.TransactionV1Envelope{
					Tx: xdr.Transaction{
						SourceAccount: genericSourceAccount,
						Memo:          xdr.Memo{},
						Operations:    []xdr.Operation{op},
					},
				},
			},
			Result: xdr.TransactionResultPair{
				Result: xdr.TransactionResult{
					Result: xdr.TransactionResultResult{
						Code:    xdr.TransactionResultCodeTxSuccess,
						Results: &changeTrustSuccess,
					},
				},
			},
			UnsafeMeta: xdr.TransactionMeta{
				V:  1,
				V1: &xdr.TransactionMetaV1{},
			},
		}

		return op, tx
	}

	opA, txA := buildTransaction(limitA)
	opB, txB := buildTransaction(limitB)
	closeMeta := makeLedgerCloseMeta()

	outputA, err := TransformOperation(opA, 0, txA, 1, closeMeta, "")
	if err != nil {
		t.Fatalf("TransformOperation(limitA) failed: %v", err)
	}
	outputB, err := TransformOperation(opB, 0, txB, 1, closeMeta, "")
	if err != nil {
		t.Fatalf("TransformOperation(limitB) failed: %v", err)
	}

	floatA, ok := outputA.OperationDetails["limit"].(float64)
	if !ok {
		t.Fatalf("expected float64 limit for outputA, got %T", outputA.OperationDetails["limit"])
	}
	floatB, ok := outputB.OperationDetails["limit"].(float64)
	if !ok {
		t.Fatalf("expected float64 limit for outputB, got %T", outputB.OperationDetails["limit"])
	}
	if floatA != floatB {
		t.Fatalf("expected float64 collision for adjacent limits, got %0.20f and %0.20f", floatA, floatB)
	}

	jsonA, err := json.Marshal(outputA.OperationDetails)
	if err != nil {
		t.Fatalf("marshal outputA details: %v", err)
	}
	jsonB, err := json.Marshal(outputB.OperationDetails)
	if err != nil {
		t.Fatalf("marshal outputB details: %v", err)
	}
	if string(jsonA) != string(jsonB) {
		t.Fatalf("expected serialized details to collide, got\nA: %s\nB: %s", jsonA, jsonB)
	}

	t.Logf("exact limit A: %s", exactA)
	t.Logf("exact limit B: %s", exactB)
	t.Logf("exported float64: %.20f", floatA)
	t.Logf("exported details JSON: %s", jsonA)
}
```

## Expected vs Actual Behavior

- **Expected**: Distinct on-chain `change_trust` limits should remain distinguishable in exported operation details, e.g. `9007199254.7409930` vs `9007199254.7409931`.
- **Actual**: Both limits export as the same floating-point JSON number, `9007199254.740993`, after lossy conversion to `float64`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the production `TransformOperation()` path and compares both the in-memory `float64` field and the serialized `details` JSON.
2. Realistic preconditions: YES — `ChangeTrustOp.Limit` is a normal XDR `Int64`, the collision threshold is below Stellar's allowed range, and this repository already uses much larger trustline limits in non-test-only transform fixtures.
3. Bug vs by-design: BUG — sibling code in the same package already uses `amount.String(op.Limit)` for exact `change_trust` formatting, and trustline exports preserve the raw limit as `int64`, so this precision loss is not required by the surrounding schema.
4. Final severity: Critical — this silently corrupts a monetary limit field in normal exports, so downstream reconciliation, compliance, or risk tooling can read a plausible but wrong asset limit.
5. In scope: YES — this is concrete financial data corruption in repository-owned ETL code.
6. Test correctness: CORRECT — the final test uses real production types, invokes the live transform, proves the exact source values differ, and shows the output collision survives serialization.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Export `change_trust` limits through an exact representation instead of `float64`, preferably reusing `amount.String(op.Limit)` or a shared exact-decimal helper for operation details. If a numeric field must remain alongside it, add a separate exact raw value and avoid making the rounded `float64` the only exported representation.
