# 008: liquidity-pool Parquet drops populated `liquidity_pool_id_strkey`

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformPool()` computes both the hex liquidity-pool ID and the canonical `L...` strkey, and the JSON export preserves both. The Parquet path silently drops the populated strkey because `PoolOutputParquet` has no `PoolIDStrkey` field and `PoolOutput.ToParquet()` therefore cannot carry it forward.

## Root Cause

The liquidity-pool JSON schema was extended with `PoolIDStrkey`, but the parallel Parquet schema and converter were not updated. As a result, every liquidity-pool row exported through `--write-parquet` loses a real identifier that the normal transform already computed.

## Reproduction

During normal operation, `export_ledger_entry_changes --write-parquet` transforms liquidity-pool ledger-entry changes with `TransformPool()` and then routes those rows through `PoolOutput.ToParquet()` into `PoolOutputParquet`. For any liquidity-pool row, the JSON output contains a non-empty `liquidity_pool_id_strkey`, while the Parquet row type has no corresponding column, so the value cannot survive conversion.

## Affected Code

- `internal/transform/liquidity_pool.go:13-89` — computes `PoolIDStrkey` and stores it on `PoolOutput`
- `internal/transform/schema.go:202-225` — JSON schema exposes `liquidity_pool_id_strkey`
- `internal/transform/schema_parquet.go:137-159` — Parquet schema omits `PoolIDStrkey`
- `internal/transform/parquet_converter.go:163-185` — `ToParquet()` converts into a struct with no destination field
- `cmd/export_ledger_entry_changes.go:321-350` — Parquet export path selects `PoolOutputParquet` for liquidity-pool rows

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestPoolOutputParquetDropsPoolIDStrkey`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./internal/transform/... -run TestPoolOutputParquetDropsPoolIDStrkey -v`
  4. Observe that `TransformPool()` populates `PoolIDStrkey`, but the converted `PoolOutputParquet` type has no `PoolIDStrkey` field.

### Test Body

```go
package transform

import (
	"reflect"
	"testing"

	"github.com/stretchr/testify/require"

	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestPoolOutputParquetDropsPoolIDStrkey(t *testing.T) {
	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}

	poolOutput, err := TransformPool(makePoolTestInput(), header)
	require.NoError(t, err)
	require.NotEmpty(t, poolOutput.PoolIDStrkey)

	parquetValue := poolOutput.ToParquet()
	parquetPool, ok := parquetValue.(PoolOutputParquet)
	require.True(t, ok, "unexpected parquet type: %T", parquetValue)

	_, hasPoolIDStrkey := reflect.TypeOf(parquetPool).FieldByName("PoolIDStrkey")
	require.False(t, hasPoolIDStrkey, "PoolOutputParquet unexpectedly has PoolIDStrkey; bug may be fixed")

	require.Equal(t, "LALS2QYAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAC2X", poolOutput.PoolIDStrkey)
	require.Equal(t, poolOutput.PoolID, parquetPool.PoolID)
}
```

## Expected vs Actual Behavior

- **Expected**: Liquidity-pool Parquet rows should preserve the same non-empty `liquidity_pool_id_strkey` that `TransformPool()` computes and the JSON export emits.
- **Actual**: `PoolOutputParquet` has no `PoolIDStrkey` field, so `PoolOutput.ToParquet()` drops the populated strkey from every Parquet row.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC runs the real `TransformPool()` path on the repository's existing liquidity-pool fixture, confirms the JSON-layer output populated `PoolIDStrkey`, and then shows the production Parquet row type has no field that could hold it.
2. Realistic preconditions: YES — any liquidity-pool ledger-entry change exported through `export_ledger_entry_changes --write-parquet` follows this exact path.
3. Bug vs by-design: BUG — the transform already treats `liquidity_pool_id_strkey` as part of the exported row, there is no comment or documentation saying Parquet should omit it, and sibling findings in this codebase treat the same JSON/Parquet identifier mismatch as a data-integrity defect.
4. Final severity: High — this is structural export corruption that silently removes a populated identifier from Parquet output, but it does not directly change monetary values.
5. In scope: YES — the production export path emits incomplete Parquet data without error.
6. Test correctness: CORRECT — the test uses production code only, establishes the populated source value first, and then inspects the concrete converted type to show the value cannot be preserved.
7. Alternative explanations: NONE — while the hex pool ID remains available, that only mitigates impact; it does not explain away the loss of a separately exported canonical identifier.
8. Novelty: NOVEL

## Suggested Fix

Add `PoolIDStrkey` to `PoolOutputParquet` and populate it in `PoolOutput.ToParquet()` so Parquet liquidity-pool rows preserve the same canonical `L...` identifier as JSON rows.
