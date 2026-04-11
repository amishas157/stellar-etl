# H002: `CONFIG_SETTING_EVICTION_ITERATOR` exports an empty shell

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --export-config-settings` encounters a
`CONFIG_SETTING_EVICTION_ITERATOR` ledger entry, the exported row should retain
the on-chain iterator state: `bucket_list_level`, `is_curr_bucket`, and
`bucket_file_offset`. JSON and Parquet consumers should see the real iterator
position instead of an all-default shell row.

## Mechanism

The current XDR union contains an `EvictionIterator` arm, but
`TransformConfigSetting()` never calls `GetEvictionIterator()`. Neither
`ConfigSettingOutput` nor `ConfigSettingOutputParquet` defines columns for the
iterator fields, so the exporter emits a plausible row with
`config_setting_id=13` and standard metadata while silently discarding all
iterator position data before serialization.

## Trigger

1. Run `export_ledger_entry_changes --export-config-settings` across a ledger
   batch that includes `CONFIG_SETTING_EVICTION_ITERATOR`.
2. Locate the emitted `config_settings` row with `config_setting_id=13`.
3. Compare it with the source XDR entry and observe that
   `bucket_list_level`, `is_curr_bucket`, and `bucket_file_offset` never appear
   in the exported JSON or Parquet row.

## Target Code

- `internal/transform/config_setting.go:13-179` — transform never reads
  `GetEvictionIterator()`
- `internal/transform/schema.go:563-627` — JSON schema lacks iterator columns
- `internal/transform/schema_parquet.go:305-369` — Parquet schema likewise has
  no iterator columns
- `internal/transform/parquet_converter.go:339-411` — converter only preserves
  fields already present in the sparse config-setting schema
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.1.0/xdr/xdr_generated.go:60965-61011`
  — upstream `EvictionIterator` arm carries three concrete fields
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.1.0/xdr/xdr_generated.go:61944-61955`
  — `GetEvictionIterator()` makes that arm available to callers

## Evidence

The transform handles adjacent config-setting arms introduced in nearby
protocol work (`ContractExecutionLanes`, `LiveSorobanStateSizeWindow`,
`ContractLedgerCostExt`) but never touches `EvictionIterator`. The output
schema has no `bucket_list_level`, `is_curr_bucket`, or `bucket_file_offset`
field, so even non-zero iterator state must collapse to a metadata-only row.

## Anti-Evidence

Some config-setting rows intentionally populate only the active union arm and
leave unrelated columns zeroed. That sparse-row design does not explain this
case, because the active arm itself has no representable destination columns in
the current exporter.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete `TransformConfigSetting()` path in `config_setting.go:13-179`. The function calls every `Get*()` accessor on the `ConfigSettingEntry` union EXCEPT `GetEvictionIterator()`. The XDR SDK at `go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775` confirms the `EvictionIterator` arm (config_setting_id=13) carries three fields: `BucketListLevel` (Uint32), `IsCurrBucket` (bool), `BucketFileOffset` (Uint64). Neither `ConfigSettingOutput` in `schema.go:563-627` nor its Parquet counterpart define destination columns for these fields. A `grep -r "EvictionIterator" internal/` returns zero results — the arm is completely unhandled.

### Code Paths Examined

- `internal/transform/config_setting.go:13-179` — `TransformConfigSetting()` calls 13 different `Get*()` accessors but never `GetEvictionIterator()`. The function produces output for every other arm.
- `internal/transform/schema.go:563-627` — `ConfigSettingOutput` struct has no `BucketListLevel`, `IsCurrBucket`, or `BucketFileOffset` fields.
- `go-stellar-sdk/xdr/xdr_generated.go` — `EvictionIterator` struct confirmed with three fields; `GetEvictionIterator()` method confirmed available; `ConfigSettingIdConfigSettingEvictionIterator = 13` confirmed.

### Findings

1. **Missing getter call**: `GetEvictionIterator()` is never invoked in `TransformConfigSetting()`, while all 13 other config setting arms have their getters called.
2. **Missing schema fields**: `ConfigSettingOutput` and `ConfigSettingOutputParquet` have no columns for the three EvictionIterator payload fields.
3. **Silent data loss**: When config_setting_id=13 appears on-chain, the transform emits a row containing only `config_setting_id=13` plus metadata (last_modified_ledger, ledger_entry_change, deleted, closed_at, ledger_sequence). All other fields are zeroed. The three actual payload fields are silently discarded with no error or warning.
4. **Distinct from sparse-row design**: The sparse-row pattern (fail/001) is intentional — unrelated columns zero out while the active arm's columns are populated. Here, the active arm itself has zero destination columns, so the row carries no payload at all.

### PoC Guidance

- **Test file**: `internal/transform/config_setting_test.go`
- **Setup**: Construct an `ingest.Change` where `LedgerEntry.Data` is a `ConfigSettingEntry` with `ConfigSettingId = ConfigSettingIdConfigSettingEvictionIterator` (13) and `EvictionIterator` populated with non-zero values (e.g., `BucketListLevel=5`, `IsCurrBucket=true`, `BucketFileOffset=12345`).
- **Steps**: Call `TransformConfigSetting(change, header)` and inspect the returned `ConfigSettingOutput`.
- **Assertion**: Demonstrate that the output contains `config_setting_id=13` but no fields reflecting the input `BucketListLevel`, `IsCurrBucket`, or `BucketFileOffset` values — all non-metadata fields are zeroed, confirming the iterator data is silently dropped.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestEvictionIteratorEmptyShell"
**Test Language**: Go

### Demonstration

The test constructs an `ingest.Change` with a `ConfigSettingEntry` whose `EvictionIterator` arm carries non-zero values (`BucketListLevel=5`, `IsCurrBucket=true`, `BucketFileOffset=12345`). After calling `TransformConfigSetting()`, the output has `config_setting_id=13` and valid metadata (LastModifiedLedger, LedgerSequence, LedgerEntryChange) but zero values for every payload field. JSON serialization confirms that `bucket_list_level`, `is_curr_bucket`, and `bucket_file_offset` are completely absent from the output schema — the EvictionIterator data is silently discarded.

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

	if !strings.Contains(jsonStr, `"config_setting_id":13`) {
		t.Errorf("JSON output should contain config_setting_id=13, got: %s", jsonStr)
	}

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

### Test Output

```
=== RUN   TestEvictionIteratorEmptyShell
    data_integrity_poc_test.go:235: BUG CONFIRMED: config_setting_id=13 (EvictionIterator) produces an empty shell row
    data_integrity_poc_test.go:236: Input EvictionIterator: BucketListLevel=5, IsCurrBucket=true, BucketFileOffset=12345
    data_integrity_poc_test.go:237: Output: All payload fields are zeroed. No JSON fields exist for bucket_list_level, is_curr_bucket, or bucket_file_offset.
    data_integrity_poc_test.go:238: The ConfigSettingOutput schema has no columns to represent EvictionIterator data — it is silently discarded.
    data_integrity_poc_test.go:252: PoC complete: TransformConfigSetting emits config_setting_id=13 with metadata but zero payload — EvictionIterator data is silently lost
--- PASS: TestEvictionIteratorEmptyShell (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.863s
```
