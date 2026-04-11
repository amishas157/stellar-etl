# 006: `CONFIG_SETTING_EVICTION_ITERATOR` exports as an empty shell row

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`TransformConfigSetting()` silently drops the entire payload of the live XDR arm `CONFIG_SETTING_EVICTION_ITERATOR`. When `export_ledger_entry_changes --export-config-settings` processes that ledger entry, it still emits a plausible row with `config_setting_id=13`, but neither JSON nor Parquet output preserves `bucket_list_level`, `is_curr_bucket`, or `bucket_file_offset`.

## Root Cause

The transformer reads many `ConfigSettingEntry` union arms, but it never calls `GetEvictionIterator()`. `ConfigSettingOutput`, `ConfigSettingOutputParquet`, and `ConfigSettingOutput.ToParquet()` also define no destination columns for the eviction-iterator fields, so the exporter serializes only metadata and silently discards the real iterator state.

## Reproduction

When a normal ledger-entry-changes export processes a `LedgerEntryTypeConfigSetting` row whose `ConfigSettingId` is `ConfigSettingIdConfigSettingEvictionIterator`, `cmd/export_ledger_entry_changes.go` always routes it through `TransformConfigSetting()` and appends the result to the `config_settings` output. The source XDR arm contains `BucketListLevel`, `IsCurrBucket`, and `BucketFileOffset`, but the exported row preserves only metadata such as `config_setting_id`, `last_modified_ledger`, `closed_at`, and `ledger_sequence`.

## Affected Code

- `internal/transform/config_setting.go:24-107` — reads many config-setting union arms but never calls `GetEvictionIterator()`
- `internal/transform/config_setting.go:116-178` — constructs the output row from existing fields only, leaving no place for the eviction-iterator payload
- `internal/transform/schema.go:563-627` — JSON schema omits `bucket_list_level`, `is_curr_bucket`, and `bucket_file_offset`
- `internal/transform/schema_parquet.go:305-369` — Parquet schema also omits those fields
- `internal/transform/parquet_converter.go:339-410` — Parquet conversion can only copy the incomplete JSON schema
- `cmd/export_ledger_entry_changes.go:245-256` — exports every config-setting entry through `TransformConfigSetting()` without filtering unsupported arms
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:60965-60968` — upstream XDR defines `EvictionIterator` with the three missing fields
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:61277` — upstream enum assigns `ConfigSettingIdConfigSettingEvictionIterator = 13`
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:61944-61955` — upstream XDR exposes `GetEvictionIterator()`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestEvictionIteratorEmptyShell`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create test file at `internal/transform/data_integrity_poc_test.go`
  3. Run: `go test ./internal/transform/... -run TestEvictionIteratorEmptyShell -v`
  4. Observe: the source arm carries `BucketListLevel=5`, `IsCurrBucket=true`, and `BucketFileOffset=12345`, but the exported row contains none of those fields.

### Test Body

```go
func TestEvictionIteratorEmptyShell(t *testing.T) {
	// 1. Construct an EvictionIterator with non-zero values
	evictionIterator := xdr.EvictionIterator{
		BucketListLevel:  xdr.Uint32(5),
		IsCurrBucket:     true,
		BucketFileOffset: xdr.Uint64(12345),
	}

	// 2. Build a ConfigSettingEntry for the EvictionIterator arm (id=13)
	configSettingEntry := xdr.ConfigSettingEntry{
		ConfigSettingId:  xdr.ConfigSettingIdConfigSettingEvictionIterator,
		EvictionIterator: &evictionIterator,
	}

	ledgerEntry := xdr.LedgerEntry{
		LastModifiedLedgerSeq: 50000000,
		Data: xdr.LedgerEntryData{
			Type:          xdr.LedgerEntryTypeConfigSetting,
			ConfigSetting: &configSettingEntry,
		},
	}

	change := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeConfigSetting,
		Pre:        nil,
		Post:       &ledgerEntry,
	}

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 100,
		},
	}

	// 3. Call the production transform
	output, err := TransformConfigSetting(change, header)
	if err != nil {
		t.Fatalf("TransformConfigSetting returned unexpected error: %v", err)
	}

	// 4. Verify the config_setting_id is correctly set to 13
	if output.ConfigSettingId != 13 {
		t.Fatalf("Expected ConfigSettingId=13, got %d", output.ConfigSettingId)
	}

	// 5. Marshal to JSON and check that the EvictionIterator fields are absent
	jsonBytes, err := json.Marshal(output)
	if err != nil {
		t.Fatalf("Failed to marshal output to JSON: %v", err)
	}
	jsonStr := string(jsonBytes)

	// The input had BucketListLevel=5, IsCurrBucket=true, BucketFileOffset=12345
	// but the output schema has no fields to represent them.
	if !strings.Contains(jsonStr, `"config_setting_id":13`) {
		t.Errorf("JSON output should contain config_setting_id=13, got: %s", jsonStr)
	}

	// Check that the iterator payload fields don't appear anywhere in the output
	hasBucketListLevel := strings.Contains(jsonStr, "bucket_list_level")
	hasIsCurrBucket := strings.Contains(jsonStr, "is_curr_bucket")
	hasBucketFileOffset := strings.Contains(jsonStr, "bucket_file_offset")

	if hasBucketListLevel || hasIsCurrBucket || hasBucketFileOffset {
		t.Errorf("Expected EvictionIterator fields to be absent from output, but found them in JSON: %s", jsonStr)
	}

	// 6. Verify that all non-metadata numeric fields are zeroed (empty shell)
	zeroChecks := []struct {
		name  string
		value interface{}
	}{
		{"ContractMaxSizeBytes", output.ContractMaxSizeBytes},
		{"LedgerMaxInstructions", output.LedgerMaxInstructions},
		{"TxMaxInstructions", output.TxMaxInstructions},
		{"FeeRatePerInstructionsIncrement", output.FeeRatePerInstructionsIncrement},
		{"TxMemoryLimit", output.TxMemoryLimit},
		{"MaxEntryTtl", output.MaxEntryTtl},
		{"EvictionScanSize", output.EvictionScanSize},
		{"LedgerMaxTxCount", output.LedgerMaxTxCount},
	}

	allZero := true
	for _, check := range zeroChecks {
		switch v := check.value.(type) {
		case uint32:
			if v != 0 {
				allZero = false
				t.Errorf("Expected %s to be 0, got %d", check.name, v)
			}
		case int64:
			if v != 0 {
				allZero = false
				t.Errorf("Expected %s to be 0, got %d", check.name, v)
			}
		case uint64:
			if v != 0 {
				allZero = false
				t.Errorf("Expected %s to be 0, got %d", check.name, v)
			}
		}
	}

	if allZero {
		t.Logf("BUG CONFIRMED: config_setting_id=13 (EvictionIterator) produces an empty shell row")
		t.Logf("Input EvictionIterator: BucketListLevel=5, IsCurrBucket=true, BucketFileOffset=12345")
		t.Logf("Output: All payload fields are zeroed. No JSON fields exist for bucket_list_level, is_curr_bucket, or bucket_file_offset.")
		t.Logf("The ConfigSettingOutput schema has no columns to represent EvictionIterator data — it is silently discarded.")
	}

	// 7. Verify metadata fields ARE populated (proving the row exists but is hollow)
	if output.LastModifiedLedger != 50000000 {
		t.Errorf("Expected LastModifiedLedger=50000000, got %d", output.LastModifiedLedger)
	}
	if output.LedgerSequence != 100 {
		t.Errorf("Expected LedgerSequence=100, got %d", output.LedgerSequence)
	}
	if output.LedgerEntryChange != 0 {
		t.Errorf("Expected LedgerEntryChange=0 (created), got %d", output.LedgerEntryChange)
	}

	t.Logf("PoC complete: TransformConfigSetting emits config_setting_id=13 with metadata but zero payload — EvictionIterator data is silently lost")
}
```

## Expected vs Actual Behavior

- **Expected**: A `CONFIG_SETTING_EVICTION_ITERATOR` export row should preserve `bucket_list_level`, `is_curr_bucket`, and `bucket_file_offset` from the source XDR arm in the exported data.
- **Actual**: The exporter emits a row with `config_setting_id=13`, but neither JSON nor Parquet contains those fields; the live iterator state is silently dropped.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC populates the real XDR arm, runs the production transformer, and shows the distinctive iterator fields never appear in the exported JSON while the row itself is emitted.
2. Realistic preconditions: YES — `export_ledger_entry_changes` sends every config-setting ledger entry through `TransformConfigSetting()` during normal operation.
3. Bug vs by-design: BUG — the sparse-union design explains zeroes in unrelated columns, but here the active arm itself has no destination columns at all, so the exporter drops the only meaningful payload for that row.
4. Final severity: High — this is structural corruption of exported protocol-configuration data, not a direct monetary miscalculation.
5. In scope: YES — it is a concrete production code path that silently exports incomplete data.
6. Test correctness: CORRECT — the test uses the real XDR type, the real transform function, and a sentinel payload that cannot be confused with metadata or a defaulted alias column.
7. Alternative explanations: NONE — there is no existing JSON or Parquet field that aliases or preserves the eviction-iterator payload.
8. Novelty: NOVEL

## Suggested Fix

Add `BucketListLevel`, `IsCurrBucket`, and `BucketFileOffset` to `ConfigSettingOutput`, `ConfigSettingOutputParquet`, and `ConfigSettingOutput.ToParquet()`, populate them from `configSetting.GetEvictionIterator()` in `TransformConfigSetting()`, and extend config-setting tests so `config_setting_id=13` exports a populated sparse-union row instead of a metadata-only shell.
