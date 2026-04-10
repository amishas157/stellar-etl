# 005: Trade parquet omits liquidity pool strkey

**Date**: 2026-04-10
**Severity**: High
**Impact**: structural data corruption
**Subsystem**: cli-commands
**Final review by**: gpt-5.4, high

## Summary

`export_trades --write-parquet` feeds the same `TradeOutput` records to JSON and parquet serialization, but only the JSON path preserves `selling_liquidity_pool_id_strkey`. For liquidity-pool trades, the transformed record contains both the raw pool ID and the canonical `LP...` strkey, yet the parquet schema and converter drop the strkey entirely.

## Root Cause

`TransformTrade` populates `TradeOutput.SellingLiquidityPoolIDStrkey` for liquidity-pool claim atoms, and the JSON schema exposes it. The parquet path uses `TradeOutputParquet` plus `TradeOutput.ToParquet()`, and both omit the field, so `export_trades` silently writes parquet rows with no column capable of carrying the populated strkey.

## Reproduction

Run `export_trades` over any ledger range containing liquidity-pool trades with `--write-parquet`. The JSON output includes `selling_liquidity_pool_id_strkey`, but the parquet export cannot emit that identifier because `TradeOutputParquet` has no such field and `WriteParquet()` serializes that struct.

## Affected Code

- `internal/transform/trade.go:TransformTrade:80-157` — liquidity-pool trades compute and assign `SellingLiquidityPoolIDStrkey`
- `internal/transform/schema.go:TradeOutput:285-312` — JSON trade schema includes `selling_liquidity_pool_id_strkey`
- `internal/transform/schema_parquet.go:TradeOutputParquet:218-244` — parquet trade schema has no `SellingLiquidityPoolIDStrkey` column
- `internal/transform/parquet_converter.go:TradeOutput.ToParquet:248-275` — parquet conversion drops the populated strkey
- `cmd/export_trades.go:tradesCmd.Run:46-70` — CLI writes JSON and, when enabled, serializes the same transformed trades through `TradeOutputParquet`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTradeParquetOmitsLiquidityPoolIDStrkey`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import (
	"encoding/json"
	"reflect"
	"strings"
	"testing"
	"time"

	"github.com/guregu/null"
)

func TestTradeParquetOmitsLiquidityPoolIDStrkey(t *testing.T) {
	tradeOutput := TradeOutput{
		Order:                        1,
		LedgerClosedAt:               time.Unix(1712700000, 0).UTC(),
		SellingAssetType:             "liquidity_pool_shares",
		SellingAmount:                0.0000123,
		BuyingAssetType:              "credit_alphanum4",
		BuyingAmount:                 0.0000456,
		PriceN:                       456,
		PriceD:                       123,
		SellingLiquidityPoolID:       null.StringFrom("0405060000000000000000000000000000000000000000000000000000000000"),
		SellingLiquidityPoolIDStrkey: null.StringFrom("LACAKBQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGOE"),
		LiquidityPoolFee:             null.IntFrom(30),
		HistoryOperationID:           101,
		TradeType:                    2,
	}

	jsonBytes, err := json.Marshal(tradeOutput)
	if err != nil {
		t.Fatalf("marshal JSON trade output: %v", err)
	}

	jsonOutput := string(jsonBytes)
	if !strings.Contains(jsonOutput, `"selling_liquidity_pool_id_strkey":"LACAKBQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGOE"`) {
		t.Fatalf("expected JSON output to contain selling_liquidity_pool_id_strkey, got %s", jsonOutput)
	}

	parquetType := reflect.TypeOf(TradeOutputParquet{})
	if _, found := parquetType.FieldByName("SellingLiquidityPoolIDStrkey"); !found {
		t.Fatalf("TradeOutputParquet has no SellingLiquidityPoolIDStrkey field, so parquet export cannot preserve JSON field %q", tradeOutput.SellingLiquidityPoolIDStrkey.String)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: When a liquidity-pool trade has `selling_liquidity_pool_id_strkey` in the transformed output, both JSON and parquet exports preserve that identifier.
- **Actual**: JSON preserves the strkey, but parquet has no column for it and silently drops it from every liquidity-pool trade row.

## Adversarial Review

1. Exercises claimed bug: YES — the test shows the production JSON path carries the field while the production parquet schema used by `WriteParquet()` cannot represent it.
2. Realistic preconditions: YES — any normal `export_trades --write-parquet` run that encounters a liquidity-pool trade hits this path.
3. Bug vs by-design: BUG — I found no CLI contract or documentation saying parquet intentionally discards populated trade fields, and `export_trades` serializes the same transformed trade objects to both formats.
4. Final severity: High — the export silently loses a non-financial identifier field that downstream systems can use to join liquidity-pool trades.
5. In scope: YES — this is a concrete production export path that emits incomplete data without error.
6. Test correctness: CORRECT — it uses production `TradeOutput`, `TradeOutputParquet`, and JSON marshalling rather than mocks or circular assertions.
7. Alternative explanations: NONE — the field is absent from the parquet schema itself, so the omission is structural.
8. Novelty: NOVEL

## Suggested Fix

Add `SellingLiquidityPoolIDStrkey string` to `TradeOutputParquet` and set it from `to.SellingLiquidityPoolIDStrkey.String` inside `TradeOutput.ToParquet()`, then keep a regression test that checks JSON/parquet parity for liquidity-pool trades.
