# 001: Float64 stroop conversion collapses distinct on-chain amounts

**Date**: 2026-04-10
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: utilities
**Final review by**: gpt-5.4, high

## Summary

`ConvertStroopValueToReal` converts exact stroop integers into `float64`, which cannot preserve single-stroop precision once values grow past about 1.07 billion real units. Adjacent on-chain amounts then collapse to the same exported number, and the ETL publishes indistinguishable balances, liabilities, reserves, and trade amounts in both JSON and Parquet outputs.

## Root Cause

The helper in `internal/utils/main.go` calls `big.Rat.Float64()` and discards the exactness result, so the export path always rounds to IEEE-754 double precision. Multiple transform outputs then store those rounded values in `float64` / Parquet `DOUBLE` fields with no parallel stroop-exact representation.

## Reproduction

Create `internal/utils/data_integrity_poc_test.go` with the PoC below and run `go test ./internal/utils/... -run TestConvertStroopValueToReal_Float64Collision -v`. The test uses adjacent stroop values and shows that the production helper returns the same exported number for both inputs.

## Affected Code

- `internal/utils/main.go:84-87` — converts stroops to `float64` via `big.Rat.Float64()`
- `internal/transform/account.go:86-90` — exports account balances and liabilities through the helper
- `internal/transform/trustline.go:68-79` — exports trustline balances and liabilities through the helper
- `internal/transform/offer.go:80-90` — exports offer amounts through the helper
- `internal/transform/trade.go:131-145` — exports trade amounts through the helper
- `internal/transform/schema.go:96-120` — JSON account schema stores these amounts as `float64`
- `internal/transform/schema.go:237-300` — trustline / trade schemas store these amounts as `float64`
- `internal/transform/schema_parquet.go:76-81` — account Parquet schema stores these amounts as `DOUBLE`
- `internal/transform/schema_parquet.go:171-233` — trustline / offer / trade Parquet schemas store these amounts as `DOUBLE`

## PoC

- **Target test file**: `internal/utils/data_integrity_poc_test.go`
- **Test name**: `TestConvertStroopValueToReal_Float64Collision`
- **Test language**: `go`
- **How to run**: Create the target test file with the test body below, then run `go test ./internal/utils/... -run TestConvertStroopValueToReal_Float64Collision -v`.

### Test Body

```go
package utils

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestConvertStroopValueToReal_Float64Collision(t *testing.T) {
	tests := []struct {
		name    string
		stroopA xdr.Int64
		stroopB xdr.Int64
	}{
		{
			name:    "first collision at ~1.07B real units",
			stroopA: 10735946655003267,
			stroopB: 10735946655003268,
		},
		{
			name:    "extreme near-max Int64",
			stroopA: 922337203685477580,
			stroopB: 922337203685477581,
		},
	}

	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			resultA := ConvertStroopValueToReal(tc.stroopA)
			resultB := ConvertStroopValueToReal(tc.stroopB)

			if resultA == resultB {
				t.Errorf("float64 collision: stroop %d and %d both produce %v", tc.stroopA, tc.stroopB, resultA)
			}
		})
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Adjacent stroop values export as distinct balances / amounts because they represent different on-chain values.
- **Actual**: `ConvertStroopValueToReal(10735946655003267)` and `ConvertStroopValueToReal(10735946655003268)` both return `1.0735946655003268e+09`; near-max inputs also collide at `9.223372036854776e+10`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the exact helper used by the affected transforms, so the collision is observed on the real export path before serialization.
2. Realistic preconditions: YES — the first collision appears at about 1.07 billion real units, which is reachable for large XLM holdings, issued-asset trustlines, and liquidity-pool reserves.
3. Bug vs by-design: BUG — the exported fields are presented as balances and amounts, not estimates, and the schemas provide no exact integer/string companion field to recover the true on-chain value.
4. Final severity: Critical — this silently corrupts financial values that downstream analytics and reconciliation systems can consume as valid numbers.
5. In scope: YES — it is a concrete ETL data-correctness defect in production code.
6. Test correctness: CORRECT — the test uses two adjacent source values and fails only when the production helper collapses them to the same output.
7. Alternative explanations: NONE — the equality arises from the `float64` conversion itself, not from formatting, mocks, or test scaffolding.
8. Novelty: NOVEL

## Suggested Fix

Preserve stroop-exact values in the exported schema instead of `float64`. The safest fixes are to export integer stroops directly, or to switch amount fields to an exact decimal/string representation and reserve display formatting for downstream consumers.
