# H002: `ConvertStroopValueToReal` rounds away valid stroops in large monetary exports

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Monetary exports should preserve exact 7-decimal stroop precision for every valid on-chain amount. For example, an account balance or trade amount of `9007199254740993` stroops should export as `900719925.4740993`, not a nearby rounded value.

## Mechanism

`utils.ConvertStroopValueToReal()` converts an `xdr.Int64` into `float64` via `big.Rat.Float64()`. Once the source amount exceeds `2^53` stroops (about `900,719,925.4740992` XLM), `float64` can no longer represent every single stroop, so the conversion silently rounds to the nearest binary float. That lossy helper is reused across live balance and amount exports, so exact Stellar values are corrupted in JSON before BigQuery ever sees them.

## Trigger

Export any ledger containing a balance, reserve, offer amount, or trade amount above `9007199254740992` stroops. A concrete example is `9007199254740993` stroops: the correct decimal is `900719925.4740993`, but the emitted JSON float will round to `900719925.4740992` (or another nearby representable float) in every field that passes through `ConvertStroopValueToReal`.

## Target Code

- `internal/utils/main.go:84-87` — `ConvertStroopValueToReal()` converts exact rational amounts to `float64`
- `internal/transform/account.go:86-90` — account balances and liabilities use the lossy helper
- `internal/transform/trustline.go:68-79` — trustline balances and liabilities use the lossy helper
- `internal/transform/offer.go:79-93` — offer amount uses the lossy helper
- `internal/transform/trade.go:131-145` — trade buy/sell amounts use the lossy helper
- `internal/transform/liquidity_pool.go:66-81` — pool share count and reserves use the lossy helper
- `internal/transform/operation.go:600-1061` — live operation `details` amounts use the same lossy helper

## Evidence

The exported schemas for these datasets store the affected monetary fields as `float64`, and every cited transformer feeds them through the same `ConvertStroopValueToReal()` helper. Stellar amounts are signed 64-bit stroop integers, while the network's real-world totals already exceed the `2^53` exact-integer limit of `float64`, so this is not a theoretical boundary: large issuer balances, XLM treasury balances, pool reserves, and trustlines can all cross it.

## Anti-Evidence

Some isolated paths do preserve an exact companion value (for example `TokenTransferOutput.AmountRaw`), which gives downstream consumers a workaround on that one dataset. But the core account, trustline, offer, trade, liquidity-pool, and operation exports cited above do not emit a lossless monetary twin, so once the float is written the exact stroop value is gone.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `ConvertStroopValueToReal` at `internal/utils/main.go:85-87` which computes `big.NewRat(int64(input), int64(10000000)).Float64()` and discards the exactness indicator via `_`. Verified all 30+ callsites across account, trustline, offer, trade, liquidity pool, and operation transformers. Each writes the lossy float64 result directly into schema fields typed as `float64` with no lossless companion field (only `TokenTransferOutput.AmountRaw` provides a string fallback, and only for that one entity type). Confirmed that the `float64` ULP at the 2^53 stroop boundary (~900M XLM) exceeds 1 stroop in real units, meaning adjacent stroop values become indistinguishable.

### Code Paths Examined

- `internal/utils/main.go:85-87` — `ConvertStroopValueToReal` uses `big.Rat.Float64()` and discards the exactness boolean; for inputs > 2^53 the nearest float64 silently rounds
- `internal/transform/account.go:88-90` — `Balance`, `BuyingLiabilities`, `SellingLiabilities` all assigned via the lossy helper; schema types are `float64` (schema.go:99-101)
- `internal/transform/trustline.go:75,78-79` — `Balance`, `BuyingLiabilities`, `SellingLiabilities` same pattern (schema.go float64 fields)
- `internal/transform/offer.go:90` — `Amount` assigned via lossy helper (schema.go float64)
- `internal/transform/trade.go:139,145` — `SellingAmount`, `BuyingAmount` both via lossy helper (schema.go:294,300 float64)
- `internal/transform/liquidity_pool.go:71,76,81` — `PoolShareCount`, `AssetAReserve`, `AssetBReserve` all via lossy helper (schema.go:208,212,217 float64)
- `internal/transform/operation.go:600-1061` — 20+ operation detail amount fields (`starting_balance`, `amount`, `source_max`, `source_amount`, `limit`, `shares`, etc.) all via lossy helper into `map[string]interface{}` which JSON-marshals as float64
- `internal/transform/schema.go:671` — Only `TokenTransferOutput.AmountRaw` provides a lossless string companion; no other entity has one

### Findings

1. **Mechanism verified**: `big.Rat.Float64()` returns the nearest representable float64 and an accuracy boolean. The function discards the boolean with `_`, providing no detection of precision loss.

2. **Precision boundary confirmed**: At magnitude ~9×10^8 (the real-unit value of 2^53 stroops), the float64 ULP is approximately 2×10^-7. Since 1 stroop = 10^-7 in real units, the ULP exceeds the stroop resolution, meaning consecutive stroop values map to the same float64.

3. **Realistic trigger**: The total XLM supply is ~50 billion XLM. The SDF's main distribution account holds tens of billions of XLM (~3×10^17 stroops), well above the 2^53 ≈ 9×10^15 threshold. Custom asset trustlines can hold up to Int64 max (~9.2×10^18 stroops). The precision boundary at ~900M XLM is routinely exceeded by major accounts.

4. **Breadth of impact**: 30+ callsites across 6 transform files feed amounts through this single lossy function. Every affected field is typed `float64` in the schema with no lossless alternative (except the single `AmountRaw` field on `TokenTransferOutput`).

5. **No guards exist**: There is no validation, logging, or error return when precision is lost. The export proceeds silently with rounded values.

### PoC Guidance

- **Test file**: `internal/utils/main_test.go` (or create new file `internal/utils/stroop_precision_test.go`)
- **Setup**: No external dependencies needed; pure unit test
- **Steps**:
  1. Call `ConvertStroopValueToReal(xdr.Int64(9007199254740993))` (which is 2^53 + 1)
  2. Multiply the result by 10,000,000 to recover the stroop value
  3. Compare against the original input
- **Assertion**: Assert that `int64(result * 10000000) != 9007199254740993` — the round-trip loses the least significant stroop. Also verify with `big.NewRat(9007199254740993, 10000000).Float64()` that the second return value (exactness) is `false`, confirming precision loss is detectable but discarded.
- **Additional test**: Call with `xdr.Int64(500000000000000000)` (~50B XLM, realistic SDF balance) and show the round-trip error is thousands of stroops.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/utils/stroop_precision_poc_test.go
**Test Name**: "TestConvertStroopValueToReal_LosesPrecisionAbove2e53" and "TestConvertStroopValueToReal_LargeRealisticBalance"
**Test Language**: Go

### Demonstration

Two tests confirm that `ConvertStroopValueToReal` silently corrupts monetary amounts. The first test shows that at the 2^53+1 boundary, a round-trip through the production function loses exactly 1 stroop, and that `big.Rat.Float64()` explicitly signals the conversion is inexact (a boolean the production code discards). The second test shows that at a realistic 50-billion-XLM balance (500,000,000,000,000,001 stroops), the function drops the fractional stroop entirely, producing a value that is provably wrong by at least 1 stroop.

### Test Body

```go
package utils

import (
	"math/big"
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestConvertStroopValueToReal_LosesPrecisionAbove2e53 demonstrates that
// ConvertStroopValueToReal silently rounds stroop values above 2^53,
// producing incorrect monetary amounts in exported data.
func TestConvertStroopValueToReal_LosesPrecisionAbove2e53(t *testing.T) {
	// 2^53 + 1 — the smallest integer that float64 cannot represent exactly
	const inputStroops xdr.Int64 = 9007199254740993

	// Run through the production code path
	realValue := ConvertStroopValueToReal(inputStroops)

	// Round-trip: multiply back to stroops
	recoveredStroops := int64(realValue * 10_000_000)

	// The round-trip should preserve the original value, but float64 loses it.
	// We assert the bug IS present: the recovered value differs from the input.
	if recoveredStroops == int64(inputStroops) {
		t.Fatalf("Expected precision loss for 2^53+1 stroops, but round-trip was exact (recovered=%d)", recoveredStroops)
	}
	t.Logf("PRECISION LOSS DEMONSTRATED: input=%d stroops, recovered=%d stroops, delta=%d stroops",
		int64(inputStroops), recoveredStroops, int64(inputStroops)-recoveredStroops)

	// Also verify that big.Rat.Float64() explicitly signals inexactness
	_, exact := big.NewRat(int64(inputStroops), 10_000_000).Float64()
	if exact {
		t.Fatal("Expected big.Rat.Float64() to report inexact conversion, but it reported exact")
	}
	t.Log("big.Rat.Float64() confirms the conversion is inexact — the discarded boolean could detect this")
}

// TestConvertStroopValueToReal_LargeRealisticBalance demonstrates precision
// loss at a realistic Stellar balance scale (~50 billion XLM + 1 stroop,
// comparable to SDF distribution account holdings).
func TestConvertStroopValueToReal_LargeRealisticBalance(t *testing.T) {
	// 50 billion XLM + 1 stroop — a realistic balance that is not a clean multiple of 10^7.
	// At this magnitude, float64 ULP in real units (~1.5×10^-5) far exceeds
	// 1 stroop in real units (10^-7), so many consecutive stroop values collapse.
	const inputStroops xdr.Int64 = 500_000_000_000_000_001

	realValue := ConvertStroopValueToReal(inputStroops)

	// Compute exact expected real value using big.Rat (no float64 loss)
	exactRat := new(big.Rat).SetFrac64(int64(inputStroops), 10_000_000)

	// Convert the float64 result back to a Rat for exact comparison
	gotRat := new(big.Rat).SetFloat64(realValue)

	if exactRat.Cmp(gotRat) == 0 {
		t.Fatalf("Expected precision loss for ~5e17 stroops, but float64 was exact")
	}

	// Compute the stroop-level error
	diff := new(big.Rat).Sub(exactRat, gotRat)
	diff.Abs(diff)
	// Convert error back to stroops (multiply by 10^7)
	diffStroops := new(big.Rat).Mul(diff, big.NewRat(10_000_000, 1))
	diffFloat, _ := diffStroops.Float64()

	t.Logf("PRECISION LOSS DEMONSTRATED: input=%d stroops (~50B XLM)", int64(inputStroops))
	t.Logf("  exact real value:  %s", exactRat.FloatString(7))
	t.Logf("  float64 real value: %.7f", realValue)
	t.Logf("  stroop-level error: %.0f stroops", diffFloat)

	if diffFloat < 1.0 {
		t.Fatal("Expected stroop-level error >= 1, but error was sub-stroop")
	}
	t.Logf("Error exceeds 1 stroop — exported value is provably wrong")
}
```

### Test Output

```
=== RUN   TestConvertStroopValueToReal_LosesPrecisionAbove2e53
    stroop_precision_poc_test.go:28: PRECISION LOSS DEMONSTRATED: input=9007199254740993 stroops, recovered=9007199254740992 stroops, delta=1 stroops
    stroop_precision_poc_test.go:36: big.Rat.Float64() confirms the conversion is inexact — the discarded boolean could detect this
--- PASS: TestConvertStroopValueToReal_LosesPrecisionAbove2e53 (0.00s)
=== RUN   TestConvertStroopValueToReal_LargeRealisticBalance
    stroop_precision_poc_test.go:67: PRECISION LOSS DEMONSTRATED: input=500000000000000001 stroops (~50B XLM)
    stroop_precision_poc_test.go:68:   exact real value:  50000000000.0000001
    stroop_precision_poc_test.go:69:   float64 real value: 50000000000.0000000
    stroop_precision_poc_test.go:70:   stroop-level error: 1 stroops
    stroop_precision_poc_test.go:75: Error exceeds 1 stroop — exported value is provably wrong
--- PASS: TestConvertStroopValueToReal_LargeRealisticBalance (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/utils	0.703s
```
