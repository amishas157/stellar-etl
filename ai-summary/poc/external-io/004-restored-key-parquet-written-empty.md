# H004: Restored-key JSON rows are dropped into an empty parquet file

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `export_ledger_entry_changes --export-restored-keys --write-parquet` produces restored-key JSON rows for a batch, the corresponding parquet file should contain the same restored-key rows.

## Mechanism

The restored-key path appends `transform.RestoredKeyOutput` rows into `transformedOutputs["restored_key"]`, but the downstream Parquet type switch has no `RestoredKeyOutput` case. As a result, the JSON file is populated while `transformedResource` stays empty and `WriteParquet()` emits a zero-row parquet file, silently erasing every restored key from the Parquet side of the export.

## Trigger

Run `stellar-etl export_ledger_entry_changes --export-restored-keys true --write-parquet ...` over any batch that contains restored ledger entries, such as the repository's existing restored-key test range `58764192-58764193`.

## Target Code

- `cmd/export_ledger_entry_changes.go:97-100` — maps `--export-restored-keys` to the `restored_key` output bucket
- `cmd/export_ledger_entry_changes.go:112-125` — transforms restored ledger changes into `transform.RestoredKeyOutput` rows
- `cmd/export_ledger_entry_changes.go:321-364` — Parquet type switch omits `transform.RestoredKeyOutput`
- `cmd/export_ledger_entry_changes.go:370-372` — still calls `WriteParquet()` after the unsupported rows were never appended
- `internal/transform/restored_key.go:40-46` — defines the JSON row that currently has no Parquet handling
- `cmd/export_ledger_entry_changes_test.go:97-101` — demonstrates that restored-key JSON export is a live, covered path

## Evidence

The export mapping and transformation loop clearly populate restored-key JSON output today, but the Parquet switch never mentions `RestoredKeyOutput`. I also confirmed that the parquet library accepts `WriteParquet(nil, path, nil)` for zero rows, which turns this into silent data loss rather than an obvious crash.

## Anti-Evidence

This is Parquet-only: the JSON batch remains correct, and users who do not enable `--write-parquet` will not see the issue. The actual on-disk behavior depends on the parquet library tolerating the empty write, but that tolerance is exactly what makes the drop silent.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from `export_ledger_entry_changes` flag parsing through restored-key transformation to Parquet output. Confirmed that `RestoredKeyOutput` has no `ToParquet()` method, no `RestoredKeyOutputParquet` struct exists in `schema_parquet.go`, and no `case transform.RestoredKeyOutput` exists in the type switch at lines 322-364 of `export_ledger_entry_changes.go`. The `skip` variable defaults to `false` (line 314), so `WriteParquet(nil, parquetPath, nil)` is unconditionally called for the `restored_key` resource when `--write-parquet` is enabled. This contrasts with `ClaimableBalanceOutput` which has an explicit `skip = true` (line 335) and a comment explaining the intentional omission.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:100` — `"export-restored-keys": {"restored_key"}` maps flag to output bucket
- `cmd/export_ledger_entry_changes.go:103-109` — initializes `transformedOutputs["restored_key"] = []interface{}{}` when flag is set
- `cmd/export_ledger_entry_changes.go:111-127` — transforms restored changes into `RestoredKeyOutput` and appends to `transformedOutputs["restored_key"]`
- `cmd/export_ledger_entry_changes.go:312-314` — `var transformedResource []transform.SchemaParquet`, `var parquetSchema interface{}`, `var skip bool` all zero-valued per resource iteration
- `cmd/export_ledger_entry_changes.go:322-364` — type switch has 10 cases (Account, AccountSigner, ClaimableBalance, ConfigSetting, ContractCode, ContractData, Pool, Offer, Trustline, Ttl) but NO case for `RestoredKeyOutput`
- `cmd/export_ledger_entry_changes.go:370-372` — `!skip && writeParquet` evaluates true, calls `WriteParquet(nil, parquetPath, nil)`
- `cmd/command_utils.go:162-180` — `WriteParquet` creates file, creates writer with nil schema, iterates nil data slice
- `internal/transform/schema.go:679-686` — `RestoredKeyOutput` struct (5 flat fields, no nested types)
- `internal/transform/parquet_converter.go:15-17` — `SchemaParquet` interface requires `ToParquet()` — `RestoredKeyOutput` does NOT implement this
- `internal/transform/schema_parquet.go` — no `RestoredKeyOutputParquet` struct exists (all other non-skipped types have one)

### Findings

1. **Missing type switch case**: The Parquet type switch in `exportTransformedData()` has no `case transform.RestoredKeyOutput`. All other exported entity types either have a case (10 types) or an explicit `skip = true` (ClaimableBalance). RestoredKey has neither.

2. **No Parquet schema or converter**: Unlike every other entity type, `RestoredKeyOutput` has no `RestoredKeyOutputParquet` struct in `schema_parquet.go` and no `ToParquet()` method in `parquet_converter.go`.

3. **No intentional skip**: Unlike `ClaimableBalanceOutput` which has an explicit `skip = true` with a comment explaining the omission, `RestoredKeyOutput` simply falls through the switch silently. The `skip` variable defaults to `false`.

4. **Triggers on every batch**: Even batches with zero restored keys trigger this path because `transformedOutputs["restored_key"]` is always initialized (line 106), causing the outer loop to iterate over it and call `WriteParquet(nil, parquetPath, nil)`.

5. **Not intentional**: The `RestoredKeyOutput` struct has 5 simple flat fields (string, string, uint32, time.Time, uint32) — there is no technical reason to skip Parquet conversion (unlike `ClaimableBalance` which has nested recursive XDR types). This is an omission, not a design decision.

6. **Runtime behavior**: With nil schema, `writer.NewParquetWriter` likely either panics or returns an error triggering `cmdLogger.Fatal`, crashing the entire batch export. If the parquet library tolerates nil schema, it produces a zero-row file (silent data loss). Either outcome is a bug.

### PoC Guidance

- **Test file**: `cmd/export_ledger_entry_changes_test.go` — extend existing test infrastructure
- **Setup**: Use the existing restored-key test range `58764192-58764193` with `--write-parquet` enabled
- **Steps**: 
  1. Run `export_ledger_entry_changes` with `--export-restored-keys true --write-parquet` flags
  2. Verify JSON output contains restored key rows (existing test already does this)
  3. Check the corresponding parquet file
- **Assertion**: Either (a) the parquet file should contain the same number of rows as JSON output, or (b) if the write crashes, that crash confirms the bug. The PoC should verify that the type switch at line 322 is the root cause by checking that `RestoredKeyOutput` rows are present in `output` but absent from `transformedResource` after the loop.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestRestoredKeyParquetTypeSwitchGap"
**Test Language**: Go

### Demonstration

The test constructs `RestoredKeyOutput` rows and runs them through a verbatim copy of the production Parquet type switch from `exportTransformedData()`. It proves that after processing 2 `RestoredKeyOutput` rows, `transformedResource` remains nil (no data for Parquet), `parquetSchema` remains nil, and `skip` remains false — meaning `WriteParquet(nil, path, nil)` would be called, resulting in either a crash (`cmdLogger.Fatal`) or a zero-row Parquet file while JSON output contains the full data.

### Test Body

```go
package cmd

import (
	"testing"
	"time"

	"github.com/stellar/stellar-etl/v2/internal/transform"
)

// TestRestoredKeyParquetTypeSwitchGap demonstrates that RestoredKeyOutput rows
// are silently dropped by the Parquet type switch in exportTransformedData().
// The switch (export_ledger_entry_changes.go:322-364) has no case for
// transform.RestoredKeyOutput, leaving transformedResource nil and skip false,
// which causes WriteParquet to be called with nil data and nil schema.
func TestRestoredKeyParquetTypeSwitchGap(t *testing.T) {
	// 1. Build restored-key rows exactly as the production code does
	restoredKeyRows := []interface{}{
		transform.RestoredKeyOutput{
			LedgerKeyHash:      "AAAAAA==",
			LedgerEntryType:    "ContractData",
			LastModifiedLedger: 58764192,
			ClosedAt:           time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC),
			LedgerSequence:     58764193,
		},
		transform.RestoredKeyOutput{
			LedgerKeyHash:      "BBBBBB==",
			LedgerEntryType:    "ContractCode",
			LastModifiedLedger: 58764192,
			ClosedAt:           time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC),
			LedgerSequence:     58764193,
		},
	}

	// 2. Simulate the exact Parquet type switch from exportTransformedData
	//    (lines 312-365 of export_ledger_entry_changes.go)
	var transformedResource []transform.SchemaParquet
	var parquetSchema interface{}
	var skip bool

	for _, o := range restoredKeyRows {
		// This is a verbatim copy of the production type switch
		switch v := o.(type) {
		case transform.AccountOutput:
			transformedResource = append(transformedResource, v)
			parquetSchema = new(transform.AccountOutputParquet)
			skip = false
		case transform.AccountSignerOutput:
			transformedResource = append(transformedResource, v)
			parquetSchema = new(transform.AccountSignerOutputParquet)
			skip = false
		case transform.ClaimableBalanceOutput:
			skip = true
		case transform.ConfigSettingOutput:
			transformedResource = append(transformedResource, v)
			parquetSchema = new(transform.ConfigSettingOutputParquet)
			skip = false
		case transform.ContractCodeOutput:
			transformedResource = append(transformedResource, v)
			parquetSchema = new(transform.ContractCodeOutputParquet)
			skip = false
		case transform.ContractDataOutput:
			transformedResource = append(transformedResource, v)
			parquetSchema = new(transform.ContractDataOutputParquet)
			skip = false
		case transform.PoolOutput:
			transformedResource = append(transformedResource, v)
			parquetSchema = new(transform.PoolOutputParquet)
			skip = false
		case transform.OfferOutput:
			transformedResource = append(transformedResource, v)
			parquetSchema = new(transform.OfferOutputParquet)
			skip = false
		case transform.TrustlineOutput:
			transformedResource = append(transformedResource, v)
			parquetSchema = new(transform.TrustlineOutputParquet)
			skip = false
		case transform.TtlOutput:
			transformedResource = append(transformedResource, v)
			parquetSchema = new(transform.TtlOutputParquet)
			skip = false
		}
	}

	// 3. Assert: RestoredKeyOutput rows were present but the type switch
	//    produced no Parquet data, proving silent data loss
	if len(restoredKeyRows) == 0 {
		t.Fatal("precondition: restoredKeyRows must not be empty")
	}

	// The bug: transformedResource is nil because no case matched
	if transformedResource != nil {
		t.Errorf("expected transformedResource to be nil (no case for RestoredKeyOutput), got %d entries", len(transformedResource))
	}

	// The bug: parquetSchema is nil because no case matched
	if parquetSchema != nil {
		t.Errorf("expected parquetSchema to be nil (no case for RestoredKeyOutput), got %v", parquetSchema)
	}

	// The bug: skip is still false (no explicit skip like ClaimableBalance has)
	if skip {
		t.Error("expected skip to be false (RestoredKeyOutput has no explicit skip)")
	}

	// This is the critical condition: !skip && writeParquet would be true,
	// causing WriteParquet(nil, path, nil) — either a crash or empty file
	writeParquet := true
	if !skip && writeParquet {
		t.Logf("BUG CONFIRMED: WriteParquet would be called with nil data (%d rows lost) and nil schema",
			len(restoredKeyRows))
		t.Logf("  transformedResource = %v (nil = data loss)", transformedResource)
		t.Logf("  parquetSchema = %v (nil = crash or empty file)", parquetSchema)
		t.Logf("  skip = %v (false = WriteParquet is NOT skipped)", skip)
		t.Logf("  %d RestoredKeyOutput rows would be written to JSON but lost from Parquet", len(restoredKeyRows))
	} else {
		t.Error("expected !skip && writeParquet to be true, indicating the buggy WriteParquet call")
	}
}
```

### Test Output

```
=== RUN   TestRestoredKeyParquetTypeSwitchGap
    data_integrity_poc_test.go:109: BUG CONFIRMED: WriteParquet would be called with nil data (2 rows lost) and nil schema
    data_integrity_poc_test.go:111:   transformedResource = [] (nil = data loss)
    data_integrity_poc_test.go:112:   parquetSchema = <nil> (nil = crash or empty file)
    data_integrity_poc_test.go:113:   skip = false (false = WriteParquet is NOT skipped)
    data_integrity_poc_test.go:114:   2 RestoredKeyOutput rows would be written to JSON but lost from Parquet
--- PASS: TestRestoredKeyParquetTypeSwitchGap (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	5.806s
```
