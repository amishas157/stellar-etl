# H004: Contract-code Parquet omits populated `ledger_key_hash_base_64`

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each `contract_code` Parquet row should preserve the same `ledger_key_hash_base_64` value that the JSON export emits, so downstream systems can recover the base64-encoded ledger key alongside the hex hash.

## Mechanism

`TransformContractCode()` computes `ledgerKeyHashBase64` and stores it on `ContractCodeOutput`, but the Parquet schema and `ToParquet()` converter never include that field. As a result, the write-parquet path silently strips a populated identifier column from every contract-code row.

## Trigger

Run `export_ledger_entry_changes --export-contract-code --write-parquet` on any ledger containing contract-code entries. The JSON rows will include `ledger_key_hash_base_64`, while the Parquet schema will omit the field entirely.

## Target Code

- `internal/transform/contract_code.go:28-41` — computes `ledgerKeyHashBase64` from the ledger key
- `internal/transform/contract_code.go:79-99` — writes `LedgerKeyHashBase64` into `ContractCodeOutput`
- `internal/transform/schema.go:548-560` — JSON schema includes `ledger_key_hash_base_64`
- `internal/transform/schema_parquet.go:284-303` — Parquet schema omits `LedgerKeyHashBase64`
- `internal/transform/parquet_converter.go:316-336` — converter omits the populated field
- `cmd/export_ledger_entry_changes.go:232-244` — contract-code rows are produced in the live export path
- `cmd/export_ledger_entry_changes.go:340-343` — Parquet export chooses `ContractCodeOutputParquet`

## Evidence

The contract-code transform follows the same pattern as contract-data: it computes both a hex hash and a base64 ledger-key encoding and stores both on the JSON output. Only the hex form exists in the Parquet schema, so format changes alone cause deterministic data loss.

## Anti-Evidence

The hex `ledger_key_hash` remains available, so consumers are not left without any identifier. But the presence of a separately computed JSON `ledger_key_hash_base_64` field strongly suggests the base64 form is a deliberate part of the export contract, not optional debug data.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full path from `TransformContractCode()` through the JSON schema to the Parquet schema and converter. The transform function at `contract_code.go:28-41` computes `ledgerKeyHashBase64` via `xdr.MarshalBase64(ledgerKey)` and stores it on the output struct at line 98. The JSON schema (`schema.go:560`) declares the field with tag `json:"ledger_key_hash_base_64"`. However, `ContractCodeOutputParquet` (`schema_parquet.go:284-303`) has no corresponding field, and `ToParquet()` (`parquet_converter.go:316-336`) never references `LedgerKeyHashBase64`. The field is silently dropped during Parquet conversion.

### Code Paths Examined

- `internal/transform/contract_code.go:28-41` — `ledgerKeyHashBase64` is computed from `xdr.MarshalBase64(ledgerKey)` and error-checked; value is always populated for valid entries
- `internal/transform/contract_code.go:79-99` — `LedgerKeyHashBase64: ledgerKeyHashBase64` assigned to `ContractCodeOutput` at line 98
- `internal/transform/schema.go:540-561` — `ContractCodeOutput` includes `LedgerKeyHashBase64 string` with JSON tag at line 560
- `internal/transform/schema_parquet.go:284-303` — `ContractCodeOutputParquet` has 18 fields but `LedgerKeyHashBase64` is absent; compare with JSON struct which has 19 data fields
- `internal/transform/parquet_converter.go:316-336` — `ToParquet()` maps all 18 Parquet fields; no reference to `cco.LedgerKeyHashBase64`
- `internal/transform/contract_code_test.go:123` — test confirms the field is populated with value `"AAAABwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"`

### Findings

The bug is confirmed. `ContractCodeOutputParquet` is missing the `LedgerKeyHashBase64` field that `ContractCodeOutput` populates. This is the same class of bug as the confirmed success finding `002-contract-event-parquet-operation-id-dropped` — a Parquet struct silently drops a field that the JSON struct includes.

Additionally, the same bug exists for `ContractDataOutput`: `schema.go:536` declares `LedgerKeyHashBase64` but `ContractDataOutputParquet` (`schema_parquet.go:260-281`) omits it, and `ContractDataOutput.ToParquet()` (`parquet_converter.go:292-314`) does not map it. This is a second instance of the same bug in a sibling entity.

### PoC Guidance

- **Test file**: `internal/transform/contract_code_test.go`
- **Setup**: Use the existing test infrastructure that constructs a `ContractCodeOutput` with a populated `LedgerKeyHashBase64` field (see line 123 of existing tests)
- **Steps**: Call `ToParquet()` on a `ContractCodeOutput` that has `LedgerKeyHashBase64` set to a non-empty string. Cast the result to `ContractCodeOutputParquet`. Verify via reflection or by checking that the Parquet struct has no `LedgerKeyHashBase64` field (demonstrating the field is dropped).
- **Assertion**: Assert that the Parquet struct type does NOT contain a `LedgerKeyHashBase64` field (confirming the schema gap), OR assert that after round-tripping through Parquet write/read the base64 value is lost while the hex hash is preserved.
