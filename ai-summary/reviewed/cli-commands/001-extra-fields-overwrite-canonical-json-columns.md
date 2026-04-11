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
