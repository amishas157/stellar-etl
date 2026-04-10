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
