# 028: liquidity-pool state export rounds exact reserves and pool-share supply

**Date**: 2026-04-14
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`TransformPool()` exports liquidity-pool reserves and total pool shares by converting exact XDR `Int64` values into `float64`. For sufficiently large pools, adjacent one-stroop states collapse to the same exported `asset_a_amount`, `asset_b_amount`, or `pool_share_count`, so downstream consumers receive a plausible but wrong pool snapshot.

The issue is not limited to in-memory rounding: the collision survives JSON serialization and is copied unchanged into Parquet output. Because the pool-state schema has no companion exact/raw reserve or share field, the lost precision cannot be recovered from the export.

## Root Cause

`TransformPool()` assigns `cp.ReserveA`, `cp.ReserveB`, and `cp.TotalPoolShares` through `utils.ConvertStroopValueToReal()`. That helper converts the exact stroop integer to the nearest IEEE-754 `float64`, which cannot preserve one-stroop increments once the scaled value grows large enough.

`PoolOutput` and `PoolOutputParquet` define those exported columns as `float64`, and `PoolOutput.ToParquet()` copies the rounded values directly into the Parquet struct. The exporter therefore bakes the precision loss into both JSON and Parquet surfaces.

## Reproduction

Any normal liquidity-pool state export can hit this path when a pool reserve or share supply grows large enough that adjacent stroop values are no longer distinguishable as `float64` after division by `1e7`. Using two otherwise identical pool entries with `ReserveA` values `90071992547409930` and `90071992547409931`, `TransformPool()` returns the same `AssetAReserve` and `json.Marshal` emits the same `asset_a_amount`.

The same collision occurs for `TotalPoolShares`, proving that distinct on-chain pool states can export as identical state rows.

## Affected Code

- `internal/transform/liquidity_pool.go:TransformPool:13-90` — converts `ReserveA`, `ReserveB`, and `TotalPoolShares` with `ConvertStroopValueToReal()`
- `internal/utils/main.go:ConvertStroopValueToReal:84-87` — rounds exact stroop integers into `float64`
- `internal/transform/schema.go:PoolOutput:202-225` — exposes `pool_share_count`, `asset_a_amount`, and `asset_b_amount` only as `float64`
- `internal/transform/schema_parquet.go:PoolOutputParquet:137-159` — mirrors the same lossy `DOUBLE` columns in Parquet
- `internal/transform/parquet_converter.go:PoolOutput.ToParquet:163-185` — copies the rounded float values into Parquet output

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestPoolReserveFloat64PrecisionLoss`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the body below
  3. Run `go test ./internal/transform/... -run TestPoolReserveFloat64PrecisionLoss -v`
  4. Observe identical exported pool values for distinct source stroop amounts

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

func TestPoolReserveFloat64PrecisionLoss(t *testing.T) {
	stroopA := xdr.Int64(90071992547409930)
	stroopB := xdr.Int64(90071992547409931)

	exactA := amount.String(stroopA)
	exactB := amount.String(stroopB)
	if exactA == exactB {
		t.Fatalf("expected exact formatter to distinguish source amounts, got %q", exactA)
	}

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue:  xdr.StellarValue{CloseTime: 1000},
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
							ReserveB:                 reserveB,
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

	t.Run("asset_a_reserve", func(t *testing.T) {
		outA, err := TransformPool(makeChange(stroopA, 100, 35), header)
		if err != nil {
			t.Fatalf("TransformPool(stroopA reserve) failed: %v", err)
		}

		outB, err := TransformPool(makeChange(stroopB, 100, 35), header)
		if err != nil {
			t.Fatalf("TransformPool(stroopB reserve) failed: %v", err)
		}

		if outA.AssetAReserve != outB.AssetAReserve {
			t.Fatalf("expected float64 collision for adjacent reserves, got %.20f and %.20f", outA.AssetAReserve, outB.AssetAReserve)
		}

		jsonA, err := json.Marshal(outA)
		if err != nil {
			t.Fatalf("marshal output A: %v", err)
		}

		jsonB, err := json.Marshal(outB)
		if err != nil {
			t.Fatalf("marshal output B: %v", err)
		}

		if string(jsonA) != string(jsonB) {
			t.Fatalf("expected identical serialized pool rows, got\nA: %s\nB: %s", jsonA, jsonB)
		}

		t.Logf("exact reserve A: %s", exactA)
		t.Logf("exact reserve B: %s", exactB)
		t.Logf("exported asset_a_amount: %.20f", outA.AssetAReserve)
		t.Logf("serialized row: %s", jsonA)
	})

	t.Run("pool_share_count", func(t *testing.T) {
		outA, err := TransformPool(makeChange(100, 100, stroopA), header)
		if err != nil {
			t.Fatalf("TransformPool(stroopA shares) failed: %v", err)
		}

		outB, err := TransformPool(makeChange(100, 100, stroopB), header)
		if err != nil {
			t.Fatalf("TransformPool(stroopB shares) failed: %v", err)
		}

		if outA.PoolShareCount != outB.PoolShareCount {
			t.Fatalf("expected float64 collision for adjacent share counts, got %.20f and %.20f", outA.PoolShareCount, outB.PoolShareCount)
		}

		t.Logf("exact share count A: %s", exactA)
		t.Logf("exact share count B: %s", exactB)
		t.Logf("exported pool_share_count: %.20f", outA.PoolShareCount)
	})
}
```

## Expected vs Actual Behavior

- **Expected**: Distinct on-chain reserve and share stroop values should remain distinguishable in exported liquidity-pool state, e.g. `9007199254.7409930` vs `9007199254.7409931`.
- **Actual**: Both states export the same rounded pool values after lossy `float64` conversion, so two different ledger entries can serialize to the same pool row.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the production `TransformPool()` path and proves the collision survives row serialization.
2. Realistic preconditions: YES — liquidity-pool reserves and pool-share supply are normal XDR `Int64` amounts, and issued-asset pools can plausibly exceed the one-stroop `float64` precision threshold.
3. Bug vs by-design: BUG — this export has no exact companion field, so using `float64` makes the authoritative pool-state snapshot silently wrong rather than merely presentation-friendly.
4. Final severity: Critical — `asset_a_amount`, `asset_b_amount`, and `pool_share_count` are financial state fields used for reserve accounting and valuation, and the export can report the wrong numeric value without error.
5. In scope: YES — this is concrete repository-owned ETL data corruption.
6. Test correctness: CORRECT — the test uses real production structs, proves the source values differ, and shows identical exported output for distinct on-chain states.
7. Alternative explanations: NONE — the collision comes from the `float64` conversion and schema, not from test scaffolding or JSON formatting alone.
8. Novelty: LIKELY NOVEL SURFACE — prior confirmed findings cover the same rounding class in other tables, but this one affects the persistent `liquidity_pools` state export.

## Suggested Fix

Export pool reserves and share supply with an exact representation instead of making `float64` the only exported value. The safest options are either:

1. store the raw stroop `Int64` values (plus optional human-readable strings), or
2. store exact decimal strings using the SDK's amount formatter and keep any derived `float64` only as a secondary convenience field.
