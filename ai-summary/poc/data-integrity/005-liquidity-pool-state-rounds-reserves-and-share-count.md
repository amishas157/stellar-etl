# H005: liquidity-pool state rounds exact reserves and pool-share supply

**Date**: 2026-04-14
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Liquidity-pool state export should preserve the exact reserve balances and total pool-share supply stored on-chain. Two valid pool ledger entries that differ by 1 stroop in `ReserveA`, `ReserveB`, or `TotalPoolShares` should produce different exported `asset_a_amount`, `asset_b_amount`, or `pool_share_count` values.

## Mechanism

`TransformPool()` converts `cp.ReserveA`, `cp.ReserveB`, and `cp.TotalPoolShares` from exact XDR `Int64` values into `float64` via `utils.ConvertStroopValueToReal()`. Once those quantities are large enough, distinct pool states collapse to the same exported numbers, which silently corrupts reserve accounting even though the source ledger entry still contains the exact stroop totals.

## Trigger

Run `TransformPool()` on two otherwise identical liquidity-pool ledger entries whose `ReserveA` or `TotalPoolShares` differ by 1 stroop above float64's exact decimal precision threshold, such as `90071992547409930` versus `90071992547409931`. Compare `asset_a_amount` or `pool_share_count`: the current JSON rows should serialize the same rounded value for both states.

## Target Code

- `internal/transform/liquidity_pool.go:66-81` — writes `PoolShareCount`, `AssetAReserve`, and `AssetBReserve` via `ConvertStroopValueToReal`
- `internal/transform/schema.go:202-216` — `PoolOutput` exposes the rounded reserve/share columns
- `internal/utils/main.go:84-87` — shared stroop helper returns `float64`
- `.../xdr/xdr_generated.go:7239-7244` — liquidity-pool reserves and total shares are exact XDR `Int64`

## Evidence

The pool transform takes exact ledger-entry quantities and loses precision only at the final ETL mapping step. These columns are authoritative state snapshots used for reserve analytics and invariant checks, so rounding away a stroop changes the exported financial state rather than merely formatting it.

## Anti-Evidence

The issue is magnitude-dependent and will not show up for typical small pools. The existing liquidity-pool findings in `ai-summary` cover missing identifiers and operation-detail rounding, not the persistent reserve/share columns of the pool-state table.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-14
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete code path from `TransformPool()` (liquidity_pool.go:66-88) through `utils.ConvertStroopValueToReal()` (utils/main.go:85-87). Confirmed that `ConvertStroopValueToReal` converts exact `xdr.Int64` values to `float64` via `big.NewRat(int64(input), 10000000).Float64()`, which is single-rounding but still lossy for stroop values exceeding 2^53 (~9×10^15). The three affected fields (`PoolShareCount`, `AssetAReserve`, `AssetBReserve`) are stored as `float64` in both JSON (`PoolOutput`) and Parquet (`PoolOutputParquet`) schemas with no companion raw/exact field, making the precision loss permanent and unrecoverable.

### Code Paths Examined

- `internal/transform/liquidity_pool.go:66-88` — `TransformPool()` constructs `PoolOutput` with `PoolShareCount: utils.ConvertStroopValueToReal(cp.TotalPoolShares)`, `AssetAReserve: utils.ConvertStroopValueToReal(cp.ReserveA)`, `AssetBReserve: utils.ConvertStroopValueToReal(cp.ReserveB)`. All three fields convert exact `xdr.Int64` to lossy `float64`.
- `internal/utils/main.go:85-87` — `ConvertStroopValueToReal(input xdr.Int64) float64` uses `big.NewRat(int64(input), int64(10000000)).Float64()`. This is single-rounding (best possible), but still loses precision for inputs > 2^53.
- `internal/transform/schema.go:202-225` — `PoolOutput` defines `PoolShareCount float64`, `AssetAReserve float64`, `AssetBReserve float64` with no companion integer/string exact fields.
- `internal/transform/schema_parquet.go:137-159` — `PoolOutputParquet` mirrors the same `float64` types (`DOUBLE` parquet type) for all three fields.
- `internal/transform/parquet_converter.go:163-186` — `PoolOutput.ToParquet()` copies `float64` values directly to Parquet struct with no additional precision.
- XDR source (`xdr_generated.go:7241-7243`) — `ReserveA Int64`, `ReserveB Int64`, `TotalPoolShares Int64` are all exact 64-bit integers.

### Findings

**Confirmed**: The mechanism is identical to the already-published success/data-integrity/024-change-trust-limit-float64-rounding finding, applied to a different export surface (pool state table vs. operation details). The `ConvertStroopValueToReal` function converts exact `xdr.Int64` reserves and shares to `float64`, losing single-stroop precision for values > 2^53 stroops (~900 million XLM equivalent).

**Key distinctions from existing findings**:
1. **Different export table**: This affects the `liquidity_pools` state table, not `history_operations.details` (024) or `history_trades` (003-reviewed) or `offers` (004-reviewed).
2. **Three affected fields**: `pool_share_count`, `asset_a_amount`, and `asset_b_amount` are all lossy.
3. **No companion exact field**: Unlike token transfers (which have `amount_raw`) or prices (which have `*_price_r`), pool state has NO fallback exact representation. The `float64` is the only exported value, making the data loss permanent.
4. **State snapshots vs. events**: Pool reserves are cumulative state snapshots used for reserve accounting and invariant checking. Corrupted reserves can cascade into derived analytics (TVL calculations, impermanent loss estimates, pool share valuations).

**Severity rationale**: Critical. Pool reserves and share counts are primary financial data. The absence of any companion exact field means downstream consumers cannot recover the true value. The threshold (~900M XLM or equivalent asset units) is within the feasible range for large issued-asset pools, though unlikely for XLM-only pools.

### PoC Guidance

- **Test file**: `internal/transform/liquidity_pool_test.go`
- **Setup**: Create two `ingest.Change` objects containing liquidity pool entries with `ReserveA` values of `90071992547409930` and `90071992547409931` (adjacent stroops above 2^53). Use identical pool parameters for everything else.
- **Steps**: Call `TransformPool()` on both changes. Compare the resulting `PoolOutput.AssetAReserve` values. Also serialize both outputs with `json.Marshal` and compare the `asset_a_amount` JSON fields.
- **Assertion**: Assert that both `AssetAReserve` float64 values are equal (demonstrating the collision). Assert that both serialized JSON `asset_a_amount` values are identical strings despite different source stroops. Repeat for `PoolShareCount` with `TotalPoolShares` values of `90071992547409930` vs `90071992547409931`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-14
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestPoolReserveFloat64PrecisionLoss"
**Test Language**: Go

### Demonstration

The test constructs two liquidity pool ledger entries differing by 1 stroop in `ReserveA` (90071992547409930 vs 90071992547409931, above 2^53) and passes both through `TransformPool()`. The resulting `AssetAReserve` float64 values are identical (9007199254.7409935), and JSON serialization produces the same `asset_a_amount` string for both. The same collision is confirmed for `PoolShareCount` via `TotalPoolShares`. This proves that the pool state export silently loses single-stroop precision for large reserves, with no companion exact field to recover the true value.

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestPoolReserveFloat64PrecisionLoss demonstrates that two liquidity pool
// entries differing by 1 stroop in ReserveA (above 2^53) produce the same
// exported float64 value, silently corrupting reserve accounting.
func TestPoolReserveFloat64PrecisionLoss(t *testing.T) {
	// Two adjacent stroop values above 2^53 where float64 cannot distinguish them
	stroopA := xdr.Int64(90071992547409930)
	stroopB := xdr.Int64(90071992547409931)

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{CloseTime: 1000},
			LedgerSeq: 10,
		},
	}

	makeChange := func(reserveA, reserveB, totalShares xdr.Int64) ingest.Change {
		entry := xdr.LedgerEntry{
			LastModifiedLedgerSeq: 30705278,
			Data: xdr.LedgerEntryData{
				Type: xdr.LedgerEntryTypeLiquidityPool,
				LiquidityPool: &xdr.LiquidityPoolEntry{
					LiquidityPoolId: xdr.PoolId{23, 45, 67},
					Body: xdr.LiquidityPoolEntryBody{
						Type: xdr.LiquidityPoolTypeLiquidityPoolConstantProduct,
						ConstantProduct: &xdr.LiquidityPoolEntryConstantProduct{
							Params: xdr.LiquidityPoolConstantProductParameters{
								AssetA: lpAssetA,
								AssetB: lpAssetB,
								Fee:    30,
							},
							ReserveA:                 reserveA,
							ReserveB:                 100,
							TotalPoolShares:          totalShares,
							PoolSharesTrustLineCount: 5,
						},
					},
				},
			},
		}
		return ingest.Change{
			ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryUpdated,
			Type:       xdr.LedgerEntryTypeLiquidityPool,
			Pre:        &entry,
			Post:       &entry,
		}
	}

	// Test 1: ReserveA precision loss
	changeA := makeChange(stroopA, 100, 35)
	changeB := makeChange(stroopB, 100, 35)

	outA, err := TransformPool(changeA, header)
	if err != nil {
		t.Fatalf("TransformPool(changeA) error: %v", err)
	}
	outB, err := TransformPool(changeB, header)
	if err != nil {
		t.Fatalf("TransformPool(changeB) error: %v", err)
	}

	// Precondition: source stroop values genuinely differ
	if stroopA == stroopB {
		t.Fatal("precondition: source stroop values must differ")
	}

	// Bug proven: both outputs produce the same float64 despite different inputs
	if outA.AssetAReserve != outB.AssetAReserve {
		t.Errorf("expected AssetAReserve collision but got different values: %v vs %v", outA.AssetAReserve, outB.AssetAReserve)
	} else {
		t.Logf("CONFIRMED: AssetAReserve precision loss — stroops %d and %d both map to %.20f",
			stroopA, stroopB, outA.AssetAReserve)
	}

	// Also confirm JSON serialization produces identical strings
	jsonA, _ := json.Marshal(outA)
	jsonB, _ := json.Marshal(outB)

	var mapA, mapB map[string]interface{}
	json.Unmarshal(jsonA, &mapA)
	json.Unmarshal(jsonB, &mapB)

	amountA := mapA["asset_a_amount"]
	amountB := mapB["asset_a_amount"]
	if amountA != amountB {
		t.Errorf("expected JSON asset_a_amount collision but values differ: %v vs %v", amountA, amountB)
	} else {
		t.Logf("CONFIRMED: JSON asset_a_amount collision — both serialize to %v despite different source stroops", amountA)
	}

	// Test 2: TotalPoolShares precision loss
	changeC := makeChange(100, 100, stroopA)
	changeD := makeChange(100, 100, stroopB)

	outC, err := TransformPool(changeC, header)
	if err != nil {
		t.Fatalf("TransformPool(changeC) error: %v", err)
	}
	outD, err := TransformPool(changeD, header)
	if err != nil {
		t.Fatalf("TransformPool(changeD) error: %v", err)
	}

	if outC.PoolShareCount != outD.PoolShareCount {
		t.Errorf("expected PoolShareCount collision but got different values: %v vs %v", outC.PoolShareCount, outD.PoolShareCount)
	} else {
		t.Logf("CONFIRMED: PoolShareCount precision loss — stroops %d and %d both map to %.20f",
			stroopA, stroopB, outC.PoolShareCount)
	}
}
```

### Test Output

```
=== RUN   TestPoolReserveFloat64PrecisionLoss
    data_integrity_poc_test.go:80: CONFIRMED: AssetAReserve precision loss — stroops 90071992547409930 and 90071992547409931 both map to 9007199254.74099349975585937500
    data_integrity_poc_test.go:97: CONFIRMED: JSON asset_a_amount collision — both serialize to 9.007199254740993e+09 despite different source stroops
    data_integrity_poc_test.go:116: CONFIRMED: PoolShareCount precision loss — stroops 90071992547409930 and 90071992547409931 both map to 9007199254.74099349975585937500
--- PASS: TestPoolReserveFloat64PrecisionLoss (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	1.004s
```
