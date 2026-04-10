# H004: Trade parquet export omits `selling_liquidity_pool_id_strkey` for pool trades

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_trades --write-parquet` exports liquidity-pool trades, parquet rows should preserve the same `selling_liquidity_pool_id_strkey` that the JSON rows expose for those trades. Downstream systems should be able to join pool trades on the canonical strkey pool identifier without having to fall back to a different export format.

## Mechanism

`TransformTrade` computes both `selling_liquidity_pool_id` and `selling_liquidity_pool_id_strkey` when the claim atom is a liquidity-pool trade. The JSON schema includes the strkey field and the tests assert it. But `TradeOutputParquet` and `TradeOutput.ToParquet()` omit that field entirely, so parquet silently drops the canonical pool identifier for every liquidity-pool trade while JSON from the same run keeps it.

## Trigger

Run `export_trades` with `--write-parquet` over a range containing liquidity-pool trades. Compare JSON and parquet output for those rows: JSON includes `selling_liquidity_pool_id_strkey`, while parquet has no column for it.

## Target Code

- `internal/transform/trade.go:80-98` — liquidity-pool trades compute both raw and strkey pool IDs
- `internal/transform/trade.go:131-157` — transformed trade rows populate `SellingLiquidityPoolIDStrkey`
- `internal/transform/schema.go:285-312` — JSON trade schema includes `selling_liquidity_pool_id_strkey`
- `internal/transform/schema_parquet.go:218-244` — parquet trade schema omits the strkey field
- `internal/transform/parquet_converter.go:248-275` — trade parquet conversion has no assignment for the strkey field
- `cmd/export_trades.go:46-70` — command writes the same transformed trades to JSON and parquet

## Evidence

The trade transform sets `SellingLiquidityPoolIDStrkey: liquidityPoolIDStrkey`, and `internal/transform/trade_test.go:789-815` asserts non-empty values for pool trades. The parquet schema ends with `seller_is_exact`, so the strkey never reaches parquet output.

## Anti-Evidence

Classic orderbook trades do not populate a liquidity-pool identifier in either format. The issue only affects liquidity-pool trade rows, but those are exactly the rows where the missing identifier matters.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformTrade` (trade.go:80-98) computes `liquidityPoolIDStrkey` via `strkey.Encode()` for liquidity pool claim atoms and assigns it to `TradeOutput.SellingLiquidityPoolIDStrkey` (trade.go:156). The JSON schema (`schema.go:311`) includes this field with tag `selling_liquidity_pool_id_strkey`, and tests (`trade_test.go:789,815`) assert non-empty strkey values for pool trades. However, `TradeOutputParquet` (`schema_parquet.go:218-244`) has no `SellingLiquidityPoolIDStrkey` field — the struct ends at `SellerIsExact`. Consequently, `ToParquet()` (`parquet_converter.go:248-274`) has no assignment for the strkey, silently dropping it from all parquet output.

### Code Paths Examined

- `internal/transform/trade.go:80-98` — liquidity pool branch computes `liquidityPoolIDStrkey` via `strkey.Encode(strkey.VersionByteLiquidityPool, id[:])` and wraps it in `null.StringFrom()`
- `internal/transform/trade.go:131-157` — `TradeOutput` struct literal assigns `SellingLiquidityPoolIDStrkey: liquidityPoolIDStrkey` at line 156
- `internal/transform/schema.go:311` — `TradeOutput` struct defines `SellingLiquidityPoolIDStrkey null.String` with JSON tag
- `internal/transform/schema_parquet.go:218-244` — `TradeOutputParquet` struct has 23 fields; `SellingLiquidityPoolIDStrkey` is absent (last field is `SellerIsExact` at line 243)
- `internal/transform/parquet_converter.go:248-274` — `ToParquet()` maps 23 fields from `TradeOutput` to `TradeOutputParquet`; no mention of `SellingLiquidityPoolIDStrkey`
- `internal/transform/trade_test.go:789,815` — test assertions confirm non-empty strkey values for pool trade test cases

### Findings

The `SellingLiquidityPoolIDStrkey` field is present in the JSON output schema and correctly populated by the transform logic, but entirely absent from the parquet schema and converter. Every liquidity pool trade exported via `--write-parquet` silently loses the canonical strkey pool identifier. The raw hex `SellingLiquidityPoolID` IS present in parquet, so the data is not completely lost — but the strkey encoding (which uses the `LP...` format with checksum) is the canonical identifier used for joins in downstream BigQuery analytics. This is a schema parity bug between JSON and Parquet output paths.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (or create a focused test in `trade_test.go`)
- **Setup**: Use the existing pool trade test fixtures from `trade_test.go` (the test cases at lines 789 and 815 that set `SellingLiquidityPoolIDStrkey`)
- **Steps**: Call `tradeOutput.ToParquet()` on a `TradeOutput` with a non-empty `SellingLiquidityPoolIDStrkey`, then assert that the resulting `TradeOutputParquet` contains the strkey value
- **Assertion**: The test will fail to compile because `TradeOutputParquet` has no `SellingLiquidityPoolIDStrkey` field — this directly demonstrates the omission. The fix requires adding the field to `TradeOutputParquet` and the corresponding assignment in `ToParquet()`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestTradeParquetOmitsLiquidityPoolIDStrkey"
**Test Language**: Go

### Demonstration

The test constructs a `TradeOutput` with a non-empty `SellingLiquidityPoolIDStrkey` (a valid LP strkey), calls the production `ToParquet()` method, and uses reflection to check whether the resulting `TradeOutputParquet` struct contains a `SellingLiquidityPoolIDStrkey` field. The field is absent, proving that every liquidity pool trade silently loses its canonical strkey identifier when exported to parquet format.

### Test Body

```go
package transform

import (
	"reflect"
	"testing"

	"github.com/guregu/null"
)

func TestTradeParquetOmitsLiquidityPoolIDStrkey(t *testing.T) {
	// Construct a TradeOutput representing a liquidity pool trade with a non-empty strkey
	tradeOutput := TradeOutput{
		Order:                        1,
		SellingLiquidityPoolID:       null.StringFrom("0405060000000000000000000000000000000000000000000000000000000000"),
		SellingLiquidityPoolIDStrkey: null.StringFrom("LACAKBQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGOE"),
		LiquidityPoolFee:             null.IntFrom(30),
		TradeType:                    2,
	}

	// Verify the JSON struct carries the strkey
	if !tradeOutput.SellingLiquidityPoolIDStrkey.Valid || tradeOutput.SellingLiquidityPoolIDStrkey.String == "" {
		t.Fatal("precondition: TradeOutput.SellingLiquidityPoolIDStrkey should be non-empty")
	}

	// Convert to parquet representation via the production code path
	parquetResult := tradeOutput.ToParquet()

	// Use reflection to check whether the parquet struct has the strkey field
	parquetVal := reflect.ValueOf(parquetResult)
	parquetType := parquetVal.Type()

	fieldFound := false
	for i := 0; i < parquetType.NumField(); i++ {
		if parquetType.Field(i).Name == "SellingLiquidityPoolIDStrkey" {
			fieldFound = true
			val := parquetVal.Field(i).String()
			if val != tradeOutput.SellingLiquidityPoolIDStrkey.String {
				t.Errorf("SellingLiquidityPoolIDStrkey value mismatch: parquet has %q, JSON has %q",
					val, tradeOutput.SellingLiquidityPoolIDStrkey.String)
			}
			break
		}
	}

	if !fieldFound {
		t.Errorf("TradeOutputParquet is missing SellingLiquidityPoolIDStrkey field; "+
			"JSON output includes selling_liquidity_pool_id_strkey=%q but parquet schema has no column for it — "+
			"liquidity pool strkey is silently dropped in parquet exports",
			tradeOutput.SellingLiquidityPoolIDStrkey.String)
	}
}
```

### Test Output

```
=== RUN   TestTradeParquetOmitsLiquidityPoolIDStrkey
    data_integrity_poc_test.go:46: TradeOutputParquet is missing SellingLiquidityPoolIDStrkey field; JSON output includes selling_liquidity_pool_id_strkey="LACAKBQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGOE" but parquet schema has no column for it — liquidity pool strkey is silently dropped in parquet exports
--- FAIL: TestTradeParquetOmitsLiquidityPoolIDStrkey (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.832s
```
