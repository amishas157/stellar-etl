# H023: Soroban contract-data/code Parquet drops `ledger_key_hash_base_64`

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When the contract-data and contract-code transforms compute the base64-encoded ledger-key XDR and expose it as `ledger_key_hash_base_64`, the Parquet exports should preserve that same value. The hashed `ledger_key_hash` is not reversible, so dropping the base64 ledger key discards exact source information that JSON still carries.

## Mechanism

Both Soroban ledger-entry transforms call `ledgerEntry.LedgerKey()` plus `xdr.MarshalBase64(...)` and store the result on their JSON/output structs. The Parquet schemas and converters omit `LedgerKeyHashBase64` entirely for both record types, so `export_ledger_entry_changes --write-parquet` silently removes the exact ledger-key encoding from contract-data and contract-code rows while still claiming success.

## Trigger

Run `export_ledger_entry_changes --export-contract-data --write-parquet` or `--export-contract-code --write-parquet` on any range with Soroban contract ledger-entry changes. The JSON rows will contain `ledger_key_hash_base_64`, while the Parquet rows will not have any column for it.

## Target Code

- `internal/transform/contract_data.go:TransformContractData:65-78` — computes the base64-encoded ledger key for contract-data rows
- `internal/transform/contract_data.go:TransformContractData:135-156` — assigns `LedgerKeyHashBase64` onto `ContractDataOutput`
- `internal/transform/contract_code.go:TransformContractCode:28-41` — computes the same base64 ledger key for contract-code rows
- `internal/transform/contract_code.go:TransformContractCode:79-99` — assigns `LedgerKeyHashBase64` onto `ContractCodeOutput`
- `internal/transform/schema.go:ContractDataOutput:515-537` — exposes the contract-data field in the JSON/output schema
- `internal/transform/schema.go:ContractCodeOutput:539-560` — exposes the contract-code field in the JSON/output schema
- `internal/transform/schema_parquet.go:ContractDataOutputParquet:260-281` — defines no Parquet column for the field
- `internal/transform/schema_parquet.go:ContractCodeOutputParquet:283-303` — defines no Parquet column for the field
- `internal/transform/parquet_converter.go:ContractDataOutput.ToParquet:292-313` — omits the field when building Parquet rows
- `internal/transform/parquet_converter.go:ContractCodeOutput.ToParquet:316-336` — omits the field when building Parquet rows
- `cmd/export_ledger_entry_changes.go:exportTransformedData:340-347` — routes both record types into the Parquet writer path

## Evidence

The transforms do real work to materialize `LedgerKeyHashBase64`: they derive the ledger key from XDR, base64-encode it, and store it in the output struct. `internal/transform/contract_data_test.go:150-164` and `internal/transform/contract_code_test.go:112-124` both assert concrete expected values for that field, showing it is part of the intended row shape. Yet neither Parquet schema has anywhere to store it.

## Anti-Evidence

The Parquet rows still retain `ledger_key_hash`, so downstream users keep a lookup hash. But a one-way hash cannot reconstruct the original ledger key bytes, so losing `ledger_key_hash_base_64` is still an irreversible data drop rather than a harmless duplication trim.
