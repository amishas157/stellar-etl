# 020: trustline Parquet drops populated `liquidity_pool_id_strkey`

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`export_ledger_entry_changes --export-trustlines --write-parquet` sends pool-share trustline rows through `TrustlineOutput.ToParquet()`, but the Parquet row type has no `LiquidityPoolIDStrkey` column. The JSON transform produces a populated canonical `L...` liquidity-pool identifier, yet the Parquet export silently drops it from every affected row.

## Root Cause

`TransformTrustline()` was updated to compute and store `LiquidityPoolIDStrkey` on `TrustlineOutput`, but the parallel Parquet schema and converter were not updated. Because `WriteParquet()` writes exactly the value returned by `ToParquet()` under the supplied `TrustlineOutputParquet` schema, the populated strkey cannot survive the conversion boundary.

## Reproduction

During normal operation, any pool-share trustline in a ledger-entry-change export follows this path: `TransformTrustline()` computes both `LiquidityPoolID` and `LiquidityPoolIDStrkey`, `export_ledger_entry_changes` selects `TrustlineOutputParquet` for the Parquet schema, and `WriteParquet()` writes `record.ToParquet()` into that schema. Since the Parquet struct has no `liquidity_pool_id_strkey` field, the exported Parquet row loses the identifier even though the JSON row for the same trustline preserves it.

## Affected Code

- `internal/transform/trustline.go:43-50` — computes the liquidity-pool StrKey for pool-share trustlines
- `internal/transform/trustline.go:68-88` — stores the populated value on `TrustlineOutput`
- `internal/transform/schema.go:237-258` — JSON trustline schema exposes `liquidity_pool_id_strkey`
- `internal/transform/schema_parquet.go:171-191` — Parquet trustline schema omits the field entirely
- `internal/transform/parquet_converter.go:199-219` — converter copies the hex pool ID but has no StrKey mapping
- `cmd/export_ledger_entry_changes.go:356-358` — export path selects `TrustlineOutputParquet` for trustline Parquet output
- `cmd/command_utils.go:162-179` — `WriteParquet()` writes `record.ToParquet()` under the supplied schema with no fallback field preservation

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTrustlineParquetDropsLiquidityPoolIDStrkey`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./internal/transform/... -run TestTrustlineParquetDropsLiquidityPoolIDStrkey -v`
  4. Observe that `TransformTrustline()` produces a non-empty `LiquidityPoolIDStrkey`, but the converted `TrustlineOutputParquet` type has no field that can carry it.

### Test Body

```go
package transform

import (
	"reflect"
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestTrustlineParquetDropsLiquidityPoolIDStrkey(t *testing.T) {
	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}

	trustlines := makeTrustlineTestInput()
	if len(trustlines) != 2 {
		t.Fatalf("expected 2 trustline fixtures, got %d", len(trustlines))
	}

	poolShareTrustline, err := TransformTrustline(trustlines[1], header)
	if err != nil {
		t.Fatalf("TransformTrustline returned unexpected error: %v", err)
	}

	if poolShareTrustline.AssetType != "pool_share" {
		t.Fatalf("expected pool_share trustline, got %q", poolShareTrustline.AssetType)
	}
	if poolShareTrustline.LiquidityPoolID == "" {
		t.Fatal("expected populated liquidity_pool_id from TransformTrustline")
	}
	if poolShareTrustline.LiquidityPoolIDStrkey == "" {
		t.Fatal("expected populated liquidity_pool_id_strkey from TransformTrustline")
	}

	parquetValue := poolShareTrustline.ToParquet()
	parquetTrustline, ok := parquetValue.(TrustlineOutputParquet)
	if !ok {
		t.Fatalf("unexpected parquet type: %T", parquetValue)
	}

	if parquetTrustline.LiquidityPoolID != poolShareTrustline.LiquidityPoolID {
		t.Fatalf("expected parquet liquidity_pool_id %q, got %q", poolShareTrustline.LiquidityPoolID, parquetTrustline.LiquidityPoolID)
	}

	if _, hasStrkeyField := reflect.TypeOf(parquetTrustline).FieldByName("LiquidityPoolIDStrkey"); hasStrkeyField {
		t.Fatal("TrustlineOutputParquet unexpectedly has LiquidityPoolIDStrkey; bug may be fixed")
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Pool-share trustline Parquet rows should preserve the same non-empty `liquidity_pool_id_strkey` that `TransformTrustline()` computes and JSON output exposes.
- **Actual**: `TrustlineOutputParquet` has no `LiquidityPoolIDStrkey` field, so `TrustlineOutput.ToParquet()` drops the populated value from every Parquet row.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC uses the production trustline fixture, runs the real `TransformTrustline()` path, and then verifies that the concrete Parquet row type cannot store the populated StrKey.
2. Realistic preconditions: YES — any ledger-entry-change export containing a pool-share trustline and `--write-parquet` reaches this exact path.
3. Bug vs by-design: BUG — the transform already treats `liquidity_pool_id_strkey` as part of the exported row, and no code or documentation states that Parquet should intentionally strip it.
4. Final severity: High — this is structural data loss in a production export format, but it does not alter financial amounts.
5. In scope: YES — the exporter emits incomplete Parquet rows without error.
6. Test correctness: CORRECT — the test uses production code only, proves the source value exists first, and then shows the writer-side schema lacks any field to preserve it.
7. Alternative explanations: NONE — the raw hex pool ID remaining available does not change the fact that a separately exported canonical identifier is silently dropped.
8. Novelty: POSSIBLE DUPLICATE — the repository summary already contains the same underlying omission under `data-transform/009`; duplicate handling belongs to the orchestrator.

## Suggested Fix

Add `LiquidityPoolIDStrkey` to `TrustlineOutputParquet` and populate it in `TrustlineOutput.ToParquet()` so trustline Parquet rows preserve the same canonical `L...` identifier as their JSON counterparts.
