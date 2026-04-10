# 009: trustline Parquet drops populated `liquidity_pool_id_strkey`

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformTrustline()` computes both the hex liquidity-pool ID and the canonical `L...` strkey for pool-share trustlines, and the JSON export preserves both. The Parquet path silently drops the populated strkey because `TrustlineOutputParquet` has no `LiquidityPoolIDStrkey` field and `TrustlineOutput.ToParquet()` therefore cannot carry it forward.

## Root Cause

The trustline JSON schema was extended with `LiquidityPoolIDStrkey`, but the parallel Parquet schema and converter were not updated. As a result, every pool-share trustline exported through `--write-parquet` loses a real identifier that the normal transform already computed.

## Reproduction

During normal operation, `export_ledger_entry_changes --write-parquet` transforms pool-share trustline ledger-entry changes with `TransformTrustline()` and then routes those rows through `TrustlineOutput.ToParquet()` into `TrustlineOutputParquet`. For any affected row, the JSON output contains a non-empty `liquidity_pool_id_strkey`, while the Parquet row type has no corresponding column, so the value cannot survive conversion.

## Affected Code

- `internal/transform/trustline.go:43-50` — computes the liquidity-pool strkey for pool-share trustlines
- `internal/transform/trustline.go:68-88` — stores the populated value on `TrustlineOutput`
- `internal/transform/schema.go:237-258` — JSON trustline schema exposes `liquidity_pool_id_strkey`
- `internal/transform/schema_parquet.go:171-191` — Parquet trustline schema omits the strkey field entirely
- `internal/transform/parquet_converter.go:199-219` — converter copies the hex pool ID but has no strkey mapping
- `cmd/export_ledger_entry_changes.go:321-358` — live Parquet export path writes `TrustlineOutputParquet`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTrustlineParquetDropsLiquidityPoolIDStrkey`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./internal/transform/... -run TestTrustlineParquetDropsLiquidityPoolIDStrkey -v`
  4. Observe that `TransformTrustline()` populates `LiquidityPoolIDStrkey`, but the converted `TrustlineOutputParquet` type has no `LiquidityPoolIDStrkey` field.

### Test Body

```go
package transform

import (
	"reflect"
	"testing"

	"github.com/stretchr/testify/require"

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
	require.Len(t, trustlines, 2)

	poolShareTrustline, err := TransformTrustline(trustlines[1], header)
	require.NoError(t, err)
	require.Equal(t, "pool_share", poolShareTrustline.AssetType)
	require.NotEmpty(t, poolShareTrustline.LiquidityPoolID)
	require.NotEmpty(t, poolShareTrustline.LiquidityPoolIDStrkey)

	parquetValue := poolShareTrustline.ToParquet()
	parquetTrustline, ok := parquetValue.(TrustlineOutputParquet)
	require.True(t, ok, "unexpected parquet type: %T", parquetValue)

	_, hasStrkeyField := reflect.TypeOf(parquetTrustline).FieldByName("LiquidityPoolIDStrkey")
	require.False(t, hasStrkeyField, "TrustlineOutputParquet unexpectedly has LiquidityPoolIDStrkey; bug may be fixed")
	require.Equal(t, poolShareTrustline.LiquidityPoolID, parquetTrustline.LiquidityPoolID)
}
```

## Expected vs Actual Behavior

- **Expected**: Pool-share trustline Parquet rows should preserve the same non-empty `liquidity_pool_id_strkey` that `TransformTrustline()` computes and the JSON export emits.
- **Actual**: `TrustlineOutputParquet` has no `LiquidityPoolIDStrkey` field, so `TrustlineOutput.ToParquet()` drops the populated strkey from every Parquet row.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC runs the real `TransformTrustline()` path on the repository's existing pool-share trustline fixture, confirms the JSON-layer output populated `LiquidityPoolIDStrkey`, and then shows the production Parquet row type has no field that could hold it.
2. Realistic preconditions: YES — any `export_ledger_entry_changes --write-parquet` run over ledgers containing pool-share trustlines follows this exact path.
3. Bug vs by-design: BUG — the transform already treats `liquidity_pool_id_strkey` as part of the exported row, there is no comment or documentation saying Parquet should omit it, and sibling confirmed findings in this repo treat the same JSON/Parquet identifier mismatch as a data-integrity defect.
4. Final severity: High — this is structural export corruption that silently removes a populated identifier from Parquet output, but it does not directly change monetary values.
5. In scope: YES — the production export path emits incomplete Parquet data without error.
6. Test correctness: CORRECT — the test uses production code only, establishes the populated source value first, and then inspects the concrete converted type to show the value cannot be preserved.
7. Alternative explanations: NONE — while the hex pool ID remains available, that only mitigates impact; it does not explain away the loss of a separately exported canonical identifier.
8. Novelty: NOVEL

## Suggested Fix

Add `LiquidityPoolIDStrkey` to `TrustlineOutputParquet` and populate it in `TrustlineOutput.ToParquet()` so Parquet trustline rows preserve the same canonical `L...` identifier as JSON rows.
