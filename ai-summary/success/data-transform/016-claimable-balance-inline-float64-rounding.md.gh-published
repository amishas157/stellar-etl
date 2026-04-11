# 016: Claimable balance inline float64 rounding

**Date**: 2026-04-11
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformClaimableBalance()` converts `ClaimableBalanceEntry.Amount` with an inline `float64(amount) / 1.0e7` expression instead of the shared stroop conversion helper used elsewhere in `internal/transform/`. For large but valid `xdr.Int64` amounts, that extra rounding changes the JSON number actually exported: the ETL writes `900719925.4740992` for an on-chain amount whose nearest-float JSON representation is `900719925.4740993`.

## Root Cause

The claimable-balance transform first rounds the integer stroop amount when casting it to `float64`, then rounds again during division by `1.0e7`. Other stroop-denominated transforms use `utils.ConvertStroopValueToReal()`, which converts from the exact rational value and avoids this extra precision loss in the exported JSON representation.

## Reproduction

Create a claimable-balance ledger entry with `Amount = 9007199254740993` stroops and run it through `TransformClaimableBalance()`. When the resulting `AssetAmount` is marshaled to JSON through the normal export path, the ETL emits `900719925.4740992`; the nearest-float conversion of the exact rational amount marshals to `900719925.4740993`.

## Affected Code

- `internal/transform/claimable_balance.go:48-66` — converts `ClaimableBalanceEntry.Amount` with inline `float64(outputAmount) / 1.0e7`
- `internal/transform/schema.go:153-169` — exposes the affected exported field as `ClaimableBalanceOutput.AssetAmount`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestClaimableBalanceAmountFloat64Rounding`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then run `go test ./internal/transform/... -run TestClaimableBalanceAmountFloat64Rounding -v`.

### Test Body

```go
package transform

import (
	"encoding/json"
	"math/big"
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestClaimableBalanceAmountFloat64Rounding(t *testing.T) {
	const largeStroops = xdr.Int64(9007199254740993)

	change := makeClaimableBalanceTestInput()
	if change.Pre == nil || change.Pre.Data.ClaimableBalance == nil {
		t.Fatal("claimable balance fixture is missing claimable balance data")
	}
	change.Pre.Data.ClaimableBalance.Amount = largeStroops

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}

	output, err := TransformClaimableBalance(change, header)
	if err != nil {
		t.Fatalf("TransformClaimableBalance returned error: %v", err)
	}

	actualJSON, err := json.Marshal(output.AssetAmount)
	if err != nil {
		t.Fatalf("marshal actual AssetAmount: %v", err)
	}

	expectedNearestFloat, _ := new(big.Rat).SetFrac64(int64(largeStroops), 10000000).Float64()
	expectedJSON, err := json.Marshal(expectedNearestFloat)
	if err != nil {
		t.Fatalf("marshal expected AssetAmount: %v", err)
	}

	inlineJSON, err := json.Marshal(float64(largeStroops) / 1.0e7)
	if err != nil {
		t.Fatalf("marshal inline AssetAmount: %v", err)
	}

	if string(expectedJSON) != "900719925.4740993" {
		t.Fatalf("sanity check failed: expected nearest-float JSON to preserve the 7th decimal digit, got %s", expectedJSON)
	}

	if string(actualJSON) != string(inlineJSON) {
		t.Fatalf("expected TransformClaimableBalance to use the inline float64 conversion path, got %s want %s", actualJSON, inlineJSON)
	}

	if string(actualJSON) != "900719925.4740992" {
		t.Fatalf("expected buggy JSON export to round down to 900719925.4740992, got %s", actualJSON)
	}

	if string(actualJSON) == string(expectedJSON) {
		t.Fatalf("expected claimable balance export to differ from the nearest-float JSON representation")
	}
}
```

## Expected vs Actual Behavior

- **Expected**: claimable-balance export should use the same nearest-float stroop conversion as the rest of the transform package, yielding JSON `900719925.4740993` for `9007199254740993` stroops.
- **Actual**: claimable-balance export bypasses the helper and emits `900719925.4740992`.

## Adversarial Review

1. Exercises claimed bug: YES — the test mutates a real `ClaimableBalanceEntry`, calls `TransformClaimableBalance()`, and checks the JSON number produced by the exported `AssetAmount`.
2. Realistic preconditions: YES — `ClaimableBalanceEntry.Amount` is `xdr.Int64`, so the trigger value is representable on chain for large native or issued-asset balances.
3. Bug vs by-design: BUG — the schema’s use of `float64` is a broader design choice, but this path is still wrong because it bypasses the shared stroop converter and produces a worse exported value than the project’s established conversion path.
4. Final severity: Critical — the field is a monetary amount, and the exporter writes a silently incorrect numeric value.
5. In scope: YES — this is a concrete data-corruption path in production transformation code.
6. Test correctness: CORRECT — the assertions verify the production code path, the actual JSON output, and the alternative nearest-float result from the exact rational value.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Replace the inline `float64(outputAmount) / 1.0e7` conversion with `utils.ConvertStroopValueToReal(balanceEntry.Amount)` so claimable balances match the rest of the transform package. If downstream consumers need exact recovery for very large amounts, add a raw stroop field alongside `AssetAmount`.
