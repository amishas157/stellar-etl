# 005: Trade Parquet drops liquidity-pool strkey

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

Liquidity-pool trades populate `selling_liquidity_pool_id_strkey` during transformation, and the JSON export preserves it. The Parquet trade schema omits that column entirely, so `export_trades --write-parquet` silently drops the canonical `L...` liquidity-pool identifier from every affected row.

## Root Cause

`TransformTrade()` computes both the hex pool ID and the strkey pool ID for liquidity-pool claim atoms and stores both on `TradeOutput`. But `TradeOutputParquet` has no `SellingLiquidityPoolIDStrkey` field, and `TradeOutput.ToParquet()` therefore has no destination for the populated value.

## Reproduction

When a ledger contains a liquidity-pool trade, `TransformTrade()` returns a `TradeOutput` whose `SellingLiquidityPoolIDStrkey` is populated. If the same row is exported through the Parquet path, the resulting `TradeOutputParquet` has no matching column, so downstream Parquet consumers cannot read the canonical strkey identifier even though JSON consumers can.

## Affected Code

- `internal/transform/trade.go:85-98` — computes the liquidity-pool strkey for pool claim atoms
- `internal/transform/trade.go:131-157` — stores the populated value on `TradeOutput`
- `internal/transform/schema.go:285-312` — JSON trade schema exposes `selling_liquidity_pool_id_strkey`
- `internal/transform/schema_parquet.go:218-244` — Parquet trade schema omits the strkey field entirely
- `internal/transform/parquet_converter.go:248-274` — converter copies the hex pool ID but has no strkey mapping
- `cmd/export_trades.go:55-70` — live Parquet export path writes `TradeOutputParquet`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTradeParquetDropsLiquidityPoolStrkey`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import (
	"reflect"
	"testing"
)

func TestTradeParquetDropsLiquidityPoolStrkey(t *testing.T) {
	transaction := makeTradeTestInput()

	trades, err := TransformTrade(4, 100, transaction, genericCloseTime)
	if err != nil {
		t.Fatalf("TransformTrade returned error: %v", err)
	}
	if len(trades) != 1 {
		t.Fatalf("expected exactly one transformed trade, got %d", len(trades))
	}

	trade := trades[0]
	if !trade.SellingLiquidityPoolIDStrkey.Valid || trade.SellingLiquidityPoolIDStrkey.String == "" {
		t.Fatal("precondition failed: TransformTrade did not populate SellingLiquidityPoolIDStrkey")
	}

	parquetRow := trade.ToParquet()
	parquetType := reflect.TypeOf(parquetRow)
	if _, ok := parquetType.FieldByName("SellingLiquidityPoolIDStrkey"); !ok {
		t.Fatalf(
			"BUG CONFIRMED: TransformTrade populated selling_liquidity_pool_id_strkey=%q, but TradeOutputParquet has no SellingLiquidityPoolIDStrkey field",
			trade.SellingLiquidityPoolIDStrkey.String,
		)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: A liquidity-pool trade exported as Parquet should retain the same `selling_liquidity_pool_id_strkey` value that `TransformTrade()` and the JSON schema already expose.
- **Actual**: The Parquet schema has no `selling_liquidity_pool_id_strkey` column, so the populated value is dropped from every affected row.

## Adversarial Review

1. Exercises claimed bug: YES — the test uses the real `TransformTrade()` path for a liquidity-pool trade, then checks the production Parquet conversion result.
2. Realistic preconditions: YES — any normal `export_trades --write-parquet` run over ledgers containing liquidity-pool trades hits this code path.
3. Bug vs by-design: BUG — the transform explicitly computes and surfaces the strkey in `TradeOutput`; there is no documented or coded rule saying the Parquet export should discard that populated identifier.
4. Final severity: High — this is silent structural data loss in one export format for a populated trade identifier field.
5. In scope: YES — this is a concrete data-integrity bug in the transform/export pipeline, not an upstream SDK or test-only issue.
6. Test correctness: CORRECT — the assertion does not depend on mocks or tautologies; the value exists before conversion and becomes unrepresentable because the Parquet type lacks the field.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Add `SellingLiquidityPoolIDStrkey` to `TradeOutputParquet`, map it in `TradeOutput.ToParquet()`, and add a regression test that compares the populated JSON and Parquet trade fields for liquidity-pool trades.
