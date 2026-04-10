# H004: Ledger-entry-change parquet drops `ledger_key_hash_base_64` for contract data and code

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --write-parquet` exports `contract_data` or `contract_code`, parquet rows should preserve both ledger-key identifiers that the JSON path already emits: `ledger_key_hash` and `ledger_key_hash_base_64`. For the repository fixtures, rows with `ledger_key_hash_base_64="AAAABg..."` or `"AAAABw..."` in JSON should expose those same values in parquet.

## Mechanism

Both `TransformContractData()` and `TransformContractCode()` compute `LedgerKeyHashBase64`, and the corresponding JSON tests assert it. But the parquet structs and `ToParquet()` converters for `ContractDataOutput` and `ContractCodeOutput` have no destination for that field. As soon as `export_ledger_entry_changes` writes parquet batches for these resource types, the base64 ledger-key payload disappears even though the JSON export for the same rows still contains it.

## Trigger

Run `export_ledger_entry_changes --write-parquet` on any ledger range containing `contract_data` or `contract_code` changes. The JSON batch includes `ledger_key_hash_base_64`, while the parquet artifact has no such column for those entity types.

## Target Code

- `internal/transform/contract_data.go:TransformContractData:65-78,135-156` — computes and emits `LedgerKeyHashBase64`
- `internal/transform/contract_code.go:TransformContractCode:28-41,79-99` — computes and emits `LedgerKeyHashBase64`
- `internal/transform/schema.go:ContractDataOutput:516-537` — JSON schema includes `ledger_key_hash_base_64`
- `internal/transform/schema.go:ContractCodeOutput:540-560` — JSON schema includes `ledger_key_hash_base_64`
- `internal/transform/contract_data_test.go:142-164` — expected contract-data output includes `LedgerKeyHashBase64`
- `internal/transform/contract_code_test.go:103-124` — expected contract-code output includes `LedgerKeyHashBase64`
- `internal/transform/schema_parquet.go:ContractDataOutputParquet:261-281` — parquet schema omits the field
- `internal/transform/schema_parquet.go:ContractCodeOutputParquet:284-303` — parquet schema omits the field
- `internal/transform/parquet_converter.go:ContractDataOutput.ToParquet:292-313` — no mapping for the field
- `internal/transform/parquet_converter.go:ContractCodeOutput.ToParquet:316-336` — no mapping for the field
- `cmd/export_ledger_entry_changes.go:exportTransformedData:321-347,370-372` — command writes parquet for both resource types

## Evidence

`ledger_key_hash_base_64` is not dead JSON-only baggage: both contract-data and contract-code transforms actively compute it from the live ledger key, and both test fixtures treat it as part of the expected export shape. The parquet path is the only place where it disappears.

## Anti-Evidence

`ledger_key_hash` is still present, so parquet consumers retain one identifier for the row. But the missing base64 field contains the serialized ledger-key form already exposed by JSON, so parquet users lose information that cannot be recovered from the hash alone.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full path from XDR ledger key extraction through JSON output and parquet serialization. Both `TransformContractData()` (contract_data.go:65-78) and `TransformContractCode()` (contract_code.go:28-41) compute `ledgerKeyHashBase64` via `xdr.MarshalBase64(ledgerKey)` and assign it to the output struct. The JSON schema structs (`ContractDataOutput` at schema.go:536, `ContractCodeOutput` at schema.go:560) include the `LedgerKeyHashBase64` field. However, both parquet schema structs (`ContractDataOutputParquet` at schema_parquet.go:261-281, `ContractCodeOutputParquet` at schema_parquet.go:284-303) completely omit this field, and both `ToParquet()` converter methods (parquet_converter.go:292-313 and 316-336) have no mapping for it.

### Code Paths Examined

- `internal/transform/contract_data.go:65-78` — computes `ledgerKeyHashBase64` from `xdr.MarshalBase64(ledgerKey)`, assigns to output struct at line 155
- `internal/transform/contract_code.go:28-41` — same computation, assigns to output struct at line 98
- `internal/transform/schema.go:536` — `ContractDataOutput.LedgerKeyHashBase64` field present with JSON tag `ledger_key_hash_base_64`
- `internal/transform/schema.go:560` — `ContractCodeOutput.LedgerKeyHashBase64` field present with JSON tag `ledger_key_hash_base_64`
- `internal/transform/schema_parquet.go:261-281` — `ContractDataOutputParquet` has 20 fields; `LedgerKeyHashBase64` is absent (last field is `ContractDataXDR`)
- `internal/transform/schema_parquet.go:284-303` — `ContractCodeOutputParquet` has 18 fields; `LedgerKeyHashBase64` is absent (last field is `NDataSegmentBytes`)
- `internal/transform/parquet_converter.go:292-313` — `ContractDataOutput.ToParquet()` maps all fields except `LedgerKeyHashBase64`
- `internal/transform/parquet_converter.go:316-336` — `ContractCodeOutput.ToParquet()` maps all fields except `LedgerKeyHashBase64`
- `internal/transform/contract_data_test.go:163` — test expects `LedgerKeyHashBase64` populated with a real base64 value
- `internal/transform/contract_code_test.go:123` — test expects `LedgerKeyHashBase64` populated with a real base64 value

### Findings

The bug is confirmed. `LedgerKeyHashBase64` is actively computed by both transform functions and tested in both test suites. The JSON output path includes the field. But the parquet output path silently drops it because:
1. Neither parquet struct defines the field
2. Neither `ToParquet()` method maps the field

This is Pattern 2 (Parquet field mapping omission). The field contains the base64-serialized ledger key, which is distinct from `LedgerKeyHash` (an FNV hash). `LedgerKeyHash` is present in parquet, but `LedgerKeyHashBase64` cannot be recovered from the hash — they carry different information. Parquet consumers lose the ability to deserialize the original ledger key.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (or nearest parquet conversion test)
- **Setup**: Create a `ContractDataOutput` and `ContractCodeOutput` with non-empty `LedgerKeyHashBase64` values (use values from existing test fixtures: `"AAAABg..."` and `"AAAABw..."`)
- **Steps**: Call `.ToParquet()` on each, cast result to the parquet struct type
- **Assertion**: Assert that the returned parquet struct contains a `LedgerKeyHashBase64` field with the same value. Currently this will fail because the field does not exist on the parquet struct. The fix requires adding `LedgerKeyHashBase64 string` to both `ContractDataOutputParquet` and `ContractCodeOutputParquet` in `schema_parquet.go`, and adding the field mapping in both `ToParquet()` methods in `parquet_converter.go`.
