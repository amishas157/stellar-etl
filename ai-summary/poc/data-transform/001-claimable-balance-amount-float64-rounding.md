# H001: Claimable-balance amounts round large stroop values through `float64`

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`ClaimableBalanceOutput.AssetAmount` should equal the exact decimal value of the on-chain `ClaimableBalanceEntry.Amount` stroop integer divided by `10^7`. For example, a balance amount of `9007199254740993` stroops should export `asset_amount = 900719925.4740993`.

## Mechanism

`TransformClaimableBalance()` converts the exact `int64` stroop amount by first casting it to `float64` and then dividing by `1.0e7`. Once the stroop value exceeds float64's exact integer range, that cast rounds before scaling, so adjacent valid claimable-balance amounts collapse to the same exported JSON number even though the underlying ledger values differ.

## Trigger

Process any claimable-balance ledger entry whose `ClaimableBalanceEntry.Amount` exceeds float64's exact-integer range, such as `9007199254740993`. The exported row will preserve the correct `balance_id` and asset metadata but emit `asset_amount = 900719925.4740992` instead of `900719925.4740993`.

## Target Code

- `internal/transform/claimable_balance.go:42-67` — direct `float64(outputAmount) / 1.0e7` conversion for `AssetAmount`
- `internal/transform/schema.go:153-169` — claimable-balance schema exposes `asset_amount` as `float64`
- `internal/transform/claimable_balance_test.go:107-131` — tests only cover small values and never exercise large-amount precision boundaries

## Evidence

This path does not reuse the already-known `ConvertStroopValueToReal()` helper; it performs its own float64 cast inline. That makes claimable balances a separate live corruption path: large claimable-balance amounts can be wrong even if downstream reviewers only checked callers of the shared helper.

## Anti-Evidence

Current fixtures use a small amount (`9990000000` stroops), so the issue stays invisible in repository tests. Consumers that recompute from raw XDR could recover, but the exported `asset_amount` field itself is silently wrong in the normal JSON output.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated. Related to `utilities/001-float64-stroop-collisions` (which covers `ConvertStroopValueToReal` callers) but this is a distinct inline code path not enumerated in that finding.

### Trace Summary

`TransformClaimableBalance()` in `claimable_balance.go:24-77` extracts `balanceEntry.Amount` (type `xdr.Int64`, which is Go `int64`) into `outputAmount` at line 48, then at line 66 performs `float64(outputAmount) / 1.0e7`. This is a two-step lossy conversion: the `float64()` cast rounds any int64 exceeding 2^53, and then the division by `1.0e7` introduces a second rounding. The result is stored in `ClaimableBalanceOutput.AssetAmount` (type `float64`, schema.go:160). Unlike `TokenTransferOutput` which preserves the exact raw amount alongside the float, `ClaimableBalanceOutput` has no stroop-exact companion field, making recovery impossible from exported data alone.

### Code Paths Examined

- `internal/transform/claimable_balance.go:48` — `outputAmount := balanceEntry.Amount` assigns `xdr.Int64` (Go `int64`) to local variable
- `internal/transform/claimable_balance.go:66` — `AssetAmount: float64(outputAmount) / 1.0e7` performs inline double-rounding conversion
- `internal/transform/schema.go:153-169` — `ClaimableBalanceOutput.AssetAmount` is `float64` with no parallel exact-integer field
- `internal/transform/claimable_balance_test.go:107-132` — test fixture uses `AssetAmount: 999` (9,990,000,000 stroops), well below the 2^53 threshold
- `internal/utils/main.go:84-87` — `ConvertStroopValueToReal` uses `big.NewRat().Float64()` (single rounding) — claimable balance does NOT use this helper
- `internal/transform/schema_parquet.go:131-134` — `ClaimableBalanceOutputParquet` is commented out/not implemented, so only JSON output is affected

### Findings

1. **Distinct code path**: The claimable balance conversion at line 66 performs `float64(int64) / 1.0e7` — a direct cast plus floating-point division. This is different from `ConvertStroopValueToReal` which uses `big.NewRat(int64, 10000000).Float64()`. The inline path double-rounds (once at cast, once at division) while the helper single-rounds (from exact rational to float64).

2. **Not covered by existing findings**: Published finding `utilities/001-float64-stroop-collisions` explicitly enumerates account, trustline, offer, and trade callers of `ConvertStroopValueToReal`. Claimable balance is absent because it doesn't use the helper. A targeted fix of that helper would leave this path broken.

3. **No recovery path**: Unlike `TokenTransferOutput` which has both `Amount` (float64) and `AmountRaw` (string), `ClaimableBalanceOutput` only has `AssetAmount` (float64). Downstream consumers cannot recover the exact stroop value from the exported JSON.

4. **Threshold**: The bug triggers when `ClaimableBalanceEntry.Amount` exceeds 2^53 = 9,007,199,254,740,992 stroops ≈ 900.7M XLM. While large for a single claimable balance, this is within the valid range of the int64 amount field and possible for large holders or issued assets.

### PoC Guidance

- **Test file**: `internal/transform/claimable_balance_test.go` (append new test)
- **Setup**: Construct a `ClaimableBalanceEntry` with `Amount = xdr.Int64(9007199254740993)` (2^53 + 1) inside a valid `ingest.Change` and `LedgerHeaderHistoryEntry`. Use the existing `makeClaimableBalanceTestInput()` as a template but override the `Amount` field.
- **Steps**: Call `TransformClaimableBalance(change, header)` and extract `output.AssetAmount`.
- **Assertion**: Assert that `output.AssetAmount != 900719925.4740993` (demonstrating precision loss). Also show that `float64(9007199254740993) / 1.0e7` produces `900719925.4740992` — a value that differs from the mathematically correct result. Compare with `big.NewRat(9007199254740993, 10000000).Float64()` to show that even single-rounding gives a different (still imprecise) result, confirming the double-rounding makes it worse.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestClaimableBalanceAmountFloat64Rounding"
**Test Language**: Go

### Demonstration

The test constructs a `ClaimableBalanceEntry` with `Amount = 9007199254740993` (2^53 + 1) and passes it through the production `TransformClaimableBalance()` function. The output `AssetAmount` is `900719925.47409916` (the double-rounded value) instead of the more precise `900719925.47409928` that `big.NewRat` produces. This confirms the inline `float64(int64) / 1.0e7` path at line 66 of `claimable_balance.go` silently corrupts large financial amounts by ~1.19e-7 XLM, and that this is a distinct code path from the `ConvertStroopValueToReal` helper.

### Test Body

```go
package transform

import (
	"math/big"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestClaimableBalanceAmountFloat64Rounding demonstrates that
// TransformClaimableBalance double-rounds large stroop amounts through float64,
// producing incorrect AssetAmount values for amounts exceeding 2^53.
func TestClaimableBalanceAmountFloat64Rounding(t *testing.T) {
	// 2^53 + 1: the smallest positive integer that float64 cannot represent exactly
	const largeStroops = int64(9007199254740993)

	// Mathematically correct result: 9007199254740993 / 10000000 = 900719925.4740993
	// Using big.Rat for exact computation
	exactRat := new(big.Rat).SetFrac64(largeStroops, 10000000)
	exactFloat, _ := exactRat.Float64()

	// The inline code path: float64(int64) / 1.0e7 — double rounds
	inlineResult := float64(largeStroops) / 1.0e7

	// Demonstrate that the inline double-rounding produces a DIFFERENT value
	// from even the single-rounding big.Rat path
	if inlineResult == exactFloat {
		t.Fatal("expected inline double-rounding to differ from big.Rat single-rounding, but they matched")
	}
	t.Logf("big.Rat single-round: %.20f", exactFloat)
	t.Logf("inline double-round:  %.20f", inlineResult)
	t.Logf("difference:           %.20e", exactFloat-inlineResult)

	// Now demonstrate the bug through the actual production code path
	ledgerEntry := xdr.LedgerEntry{
		Ext: xdr.LedgerEntryExt{
			V: 1,
			V1: &xdr.LedgerEntryExtensionV1{
				SponsoringId: &xdr.AccountId{
					Type:    0,
					Ed25519: &xdr.Uint256{1, 2, 3, 4, 5, 6, 7, 8, 9},
				},
				Ext: xdr.LedgerEntryExtensionV1Ext{
					V: 1,
				},
			},
		},
		LastModifiedLedgerSeq: 30705278,
		Data: xdr.LedgerEntryData{
			Type: xdr.LedgerEntryTypeClaimableBalance,
			ClaimableBalance: &xdr.ClaimableBalanceEntry{
				BalanceId: genericClaimableBalance,
				Claimants: []xdr.Claimant{
					{
						Type: 0,
						V0: &xdr.ClaimantV0{
							Destination: testAccount1ID,
						},
					},
				},
				Asset: xdr.Asset{
					Type: xdr.AssetTypeAssetTypeCreditAlphanum12,
					AlphaNum12: &xdr.AlphaNum12{
						AssetCode: xdr.AssetCode12{1, 2, 3, 4, 5, 6, 7, 8, 9},
						Issuer:    testAccount3ID,
					},
				},
				Amount: xdr.Int64(largeStroops),
				Ext: xdr.ClaimableBalanceEntryExt{
					V: 1,
					V1: &xdr.ClaimableBalanceEntryExtensionV1{
						Ext: xdr.ClaimableBalanceEntryExtensionV1Ext{
							V: 1,
						},
						Flags: 10,
					},
				},
			},
		},
	}

	change := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryRemoved,
		Type:       xdr.LedgerEntryTypeClaimableBalance,
		Pre:        &ledgerEntry,
		Post:       nil,
	}

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

	// The production code uses float64(outputAmount) / 1.0e7 which double-rounds.
	// This produces 900719925.4740992 instead of the more accurate 900719925.4740993.
	// Assert that the output matches the double-rounded (wrong) value, proving the bug.
	if output.AssetAmount != inlineResult {
		t.Errorf("expected AssetAmount to match inline double-rounded value %v, got %v",
			inlineResult, output.AssetAmount)
	}

	// The output does NOT match the more precise single-rounding value
	if output.AssetAmount == exactFloat {
		t.Error("AssetAmount unexpectedly matches the big.Rat precise value — bug may be fixed")
	}

	t.Logf("Production AssetAmount: %.20f", output.AssetAmount)
	t.Logf("Precise (big.Rat):      %.20f", exactFloat)
	t.Logf("Input stroops:          %d", largeStroops)
	t.Logf("BUG CONFIRMED: AssetAmount loses precision due to float64 double-rounding")
}
```

### Test Output

```
=== RUN   TestClaimableBalanceAmountFloat64Rounding
    data_integrity_poc_test.go:31: big.Rat single-round: 900719925.47409927845001220703
    data_integrity_poc_test.go:32: inline double-round:  900719925.47409915924072265625
    data_integrity_poc_test.go:33: difference:           1.19209289550781250000e-07
    data_integrity_poc_test.go:117: Production AssetAmount: 900719925.47409915924072265625
    data_integrity_poc_test.go:118: Precise (big.Rat):      900719925.47409927845001220703
    data_integrity_poc_test.go:119: Input stroops:          9007199254740993
    data_integrity_poc_test.go:120: BUG CONFIRMED: AssetAmount loses precision due to float64 double-rounding
--- PASS: TestClaimableBalanceAmountFloat64Rounding (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.807s
```
