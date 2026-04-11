# H001: `--extra-fields` can overwrite canonical JSON export columns

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When an operator supplies `--extra-fields`, stellar-etl should append metadata columns without changing any canonical export field already produced from ledger data. If an extra-field key collides with a reserved column such as `ledger_sequence`, `id`, or `fee_charged`, the command should reject it or preserve the on-chain value.

## Mechanism

`ExportEntry` marshals the transformed row into a generic map and then blindly writes every `extra` key back into that map with `i[k] = v`. Because this happens after the canonical fields are decoded, a colliding extra-field silently replaces the real exported value and also changes its JSON type to `string`, producing plausible-but-wrong JSON rows while parquet output from the same run remains unmodified.

## Trigger

Run any JSON-exporting command with a colliding metadata flag such as `--extra-fields ledger_sequence=0` or `--extra-fields fee_charged=9999999`. The resulting JSON rows will contain the injected string value instead of the real ledger-derived field.

## Target Code

- `cmd/command_utils.go:ExportEntry:55-86` — extra fields are merged after the canonical row is decoded
- `internal/utils/main.go:AddCommonFlags:232-245` — `extra-fields` is exposed as generic metadata with no reserved-key guard
- `cmd/export_transactions.go:19-24` — transaction export forwards `commonArgs.Extra` into `ExportEntry`
- `README.md:155-165` — documentation describes extra fields as appended metadata

## Evidence

The merge loop in `ExportEntry` does not check whether `k` already exists in the decoded row map and always overwrites the existing value. Because `extra-fields` is a `StringToString` flag, any replaced numeric or boolean column is serialized back out as a string literal, creating a silent JSON/parquet divergence for the same export run.

## Anti-Evidence

This only affects operators who choose a colliding extra-field key; unique metadata keys still behave as intended. Parquet output is unaffected because the extra-field merge happens only in the JSON helper.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from flag registration (`internal/utils/main.go:237`) through flag parsing (`internal/utils/main.go:341,481`) to every export command that passes `commonArgs.Extra` to `ExportEntry` (all 10 export commands). Confirmed that `ExportEntry` at `cmd/command_utils.go:69-71` unconditionally overwrites map keys with no collision check. Neither the flag registration nor the parsing path includes any reserved-key validation.

### Code Paths Examined

- `internal/utils/main.go:237` — `StringToStringP("extra-fields", ...)` registered with no key validation
- `internal/utils/main.go:341-344` — `GetStringToString("extra-fields")` parsed without reserved-key guard (common flags path)
- `internal/utils/main.go:481-484` — same parsing in archive flags path
- `cmd/command_utils.go:55-71` — `ExportEntry` marshals entry to JSON, decodes into `map[string]interface{}`, then `for k, v := range extra { i[k] = v }` overwrites any existing key
- `cmd/export_transactions.go:43` — passes `commonArgs.Extra` directly to `ExportEntry`
- `cmd/export_ledger_entry_changes.go:283,316` — async batch path also passes `extra` through `exportTransformedData` to `ExportEntry`
- `cmd/export_ledgers.go:50`, `cmd/export_operations.go:43`, `cmd/export_effects.go:45`, `cmd/export_trades.go:47`, `cmd/export_assets.go:59`, `cmd/export_contract_events.go:43`, `cmd/export_token_transfers.go:47`, `cmd/export_ledger_transaction.go:42` — all 10 export commands affected

### Findings

1. **No collision guard exists anywhere in the pipeline.** The `extra-fields` flag accepts arbitrary string keys without validation. The merge in `ExportEntry` unconditionally overwrites, meaning any canonical field (including financial fields like `fee_charged`, `max_fee`, amounts) can be silently replaced.

2. **Type corruption compounds the issue.** Because `extra-fields` is a `StringToString` Cobra flag, all values are strings. When a numeric canonical field like `fee_charged` (normally a JSON number) is overwritten, it becomes a JSON string `"9999999"` instead of a JSON number `9999999`. Downstream JSON parsers expecting numeric types would either fail or silently misinterpret the value.

3. **JSON/Parquet divergence.** The overwrite only affects JSON output because `WriteParquet` operates on the typed Go structs directly. A dual-format export run (`--write-parquet` + JSON) would produce different values for the same field in the same row — JSON contains the overwritten string value while Parquet contains the correct numeric value.

4. **All 10 export commands are affected.** Every export command in `cmd/` passes `commonArgs.Extra` (or `cmdArgs.Extra`) to `ExportEntry` without pre-filtering.

### PoC Guidance

- **Test file**: `cmd/command_utils_test.go` (or create if none exists)
- **Setup**: Create a mock `LedgerOutput` or `TransactionOutput` struct with a known `ledger_sequence` value (e.g., 42). Prepare an `extra` map with a colliding key: `map[string]string{"ledger_sequence": "0"}`.
- **Steps**: Call `ExportEntry(output, tmpFile, extra)`. Read back the JSON from the file. Unmarshal into `map[string]interface{}`.
- **Assertion**: Assert that `result["ledger_sequence"]` equals the string `"0"` (demonstrating the overwrite) instead of the JSON number `42`. Additionally assert the type changed from `json.Number` to `string`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestExtraFieldsOverwriteCanonicalColumns"
**Test Language**: Go

### Demonstration

The test creates a `LedgerOutput` with known canonical values (sequence=42, base_fee=100, id=999, ledger_hash="abc123") and calls `ExportEntry` with colliding extra-fields. The output JSON contains the injected string values instead of the original ledger-derived values, confirming that `ExportEntry` unconditionally overwrites canonical columns. The type also changes from `json.Number` to `string`, demonstrating silent type corruption.

### Test Body

```go
package cmd

import (
	"bytes"
	"encoding/json"
	"os"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
)

// TestExtraFieldsOverwriteCanonicalColumns demonstrates that ExportEntry allows
// --extra-fields to silently overwrite canonical JSON columns. A colliding key
// replaces the real ledger-derived value and changes the JSON type from number
// to string.
func TestExtraFieldsOverwriteCanonicalColumns(t *testing.T) {
	// 1. Construct a LedgerOutput with known canonical values.
	entry := transform.LedgerOutput{
		Sequence:        42,
		LedgerHash:      "abc123",
		BaseFee:         100,
		ProtocolVersion: 21,
		LedgerID:        999,
	}

	// 2. Prepare extra-fields that collide with canonical columns.
	extra := map[string]string{
		"sequence":    "0",
		"base_fee":    "9999999",
		"id":          "INJECTED",
		"ledger_hash": "OVERWRITTEN",
	}

	// 3. Call ExportEntry with a temp file.
	tmpFile, err := os.CreateTemp("", "poc-extra-fields-*.json")
	if err != nil {
		t.Fatalf("failed to create temp file: %v", err)
	}
	defer os.Remove(tmpFile.Name())
	defer tmpFile.Close()

	_, err = ExportEntry(entry, tmpFile, extra)
	if err != nil {
		t.Fatalf("ExportEntry returned error: %v", err)
	}

	// 4. Read back the JSON output.
	tmpFile.Seek(0, 0)
	var buf bytes.Buffer
	buf.ReadFrom(tmpFile)

	// 5. Decode as generic map using UseNumber to distinguish types.
	decoder := json.NewDecoder(bytes.NewReader(buf.Bytes()))
	decoder.UseNumber()
	result := map[string]interface{}{}
	if err := decoder.Decode(&result); err != nil {
		t.Fatalf("failed to decode JSON output: %v", err)
	}

	// 6. Assert: canonical fields were overwritten by extra-fields.
	//    If the bug exists, "sequence" will be the string "0" instead of json.Number "42".

	// Check sequence was overwritten
	seq := result["sequence"]
	seqStr, isString := seq.(string)
	if !isString {
		t.Fatalf("Expected 'sequence' to be overwritten to a string, but got type %T value %v", seq, seq)
	}
	if seqStr != "0" {
		t.Errorf("Expected 'sequence' = \"0\" (injected), got %q", seqStr)
	}
	t.Logf("BUG CONFIRMED: 'sequence' overwritten from json.Number(42) to string \"0\"")

	// Check base_fee was overwritten
	bf := result["base_fee"]
	bfStr, isString := bf.(string)
	if !isString {
		t.Fatalf("Expected 'base_fee' to be overwritten to a string, but got type %T value %v", bf, bf)
	}
	if bfStr != "9999999" {
		t.Errorf("Expected 'base_fee' = \"9999999\" (injected), got %q", bfStr)
	}
	t.Logf("BUG CONFIRMED: 'base_fee' overwritten from json.Number(100) to string \"9999999\"")

	// Check id was overwritten
	id := result["id"]
	idStr, isString := id.(string)
	if !isString {
		t.Fatalf("Expected 'id' to be overwritten to a string, but got type %T value %v", id, id)
	}
	if idStr != "INJECTED" {
		t.Errorf("Expected 'id' = \"INJECTED\", got %q", idStr)
	}
	t.Logf("BUG CONFIRMED: 'id' overwritten from json.Number(999) to string \"INJECTED\"")

	// Check ledger_hash was overwritten (string->string, but still wrong value)
	lh := result["ledger_hash"]
	lhStr, isString := lh.(string)
	if !isString {
		t.Fatalf("Expected 'ledger_hash' to be a string, but got type %T", lh)
	}
	if lhStr != "OVERWRITTEN" {
		t.Errorf("Expected 'ledger_hash' = \"OVERWRITTEN\" (injected), got %q", lhStr)
	}
	t.Logf("BUG CONFIRMED: 'ledger_hash' overwritten from \"abc123\" to \"OVERWRITTEN\"")

	// Summary: all canonical fields were silently replaced
	t.Log("SUMMARY: ExportEntry allows --extra-fields to silently overwrite any canonical JSON column")
}
```

### Test Output

```
=== RUN   TestExtraFieldsOverwriteCanonicalColumns
    data_integrity_poc_test.go:72: BUG CONFIRMED: 'sequence' overwritten from json.Number(42) to string "0"
    data_integrity_poc_test.go:83: BUG CONFIRMED: 'base_fee' overwritten from json.Number(100) to string "9999999"
    data_integrity_poc_test.go:94: BUG CONFIRMED: 'id' overwritten from json.Number(999) to string "INJECTED"
    data_integrity_poc_test.go:105: BUG CONFIRMED: 'ledger_hash' overwritten from "abc123" to "OVERWRITTEN"
    data_integrity_poc_test.go:108: SUMMARY: ExportEntry allows --extra-fields to silently overwrite any canonical JSON column
--- PASS: TestExtraFieldsOverwriteCanonicalColumns (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	5.713s
```
