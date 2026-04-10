# H003: Contract-data Parquet omits populated `ledger_key_hash_base_64`

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each `contract_data` Parquet row should preserve the same `ledger_key_hash_base_64` value that `TransformContractData()` computes and the JSON export emits. The base64 ledger-key encoding should remain available for downstream consumers that reconstruct or compare the original XDR ledger key.

## Mechanism

`TransformContractData()` marshals the ledger key to base64 and stores it on `ContractDataOutput.LedgerKeyHashBase64`. The Parquet schema and converter never carry that field forward, so `export_ledger_entry_changes --export-contract-data --write-parquet` silently drops a populated identifier column from every non-nonce contract-data row.

## Trigger

Run `export_ledger_entry_changes --export-contract-data --write-parquet` over any ledger that contains contract-data entries. The JSON output will include `ledger_key_hash_base_64`, but the Parquet file will have no such column and therefore no way to retain the computed value.

## Target Code

- `internal/transform/contract_data.go:65-78` — computes `ledgerKeyHashBase64` from the ledger key XDR
- `internal/transform/contract_data.go:135-156` — writes `LedgerKeyHashBase64` into `ContractDataOutput`
- `internal/transform/schema.go:530-536` — JSON schema exposes `ledger_key_hash_base_64`
- `internal/transform/schema_parquet.go:261-281` — Parquet schema omits `LedgerKeyHashBase64`
- `internal/transform/parquet_converter.go:292-313` — converter omits the populated field
- `cmd/export_ledger_entry_changes.go:212-230` — contract-data rows are produced in the live export path
- `cmd/export_ledger_entry_changes.go:321-347` — Parquet export chooses `ContractDataOutputParquet` for those rows

## Evidence

The transformer explicitly computes both `LedgerKeyHash` and `LedgerKeyHashBase64`, and both are kept on the JSON output struct. But only the hex hash survives the Parquet schema/converter pair, so the base64 representation is deterministically lost whenever the parquet path is used.

## Anti-Evidence

Consumers can still access the hex `ledger_key_hash`, so one identifier remains. But the code intentionally computes and exports the base64 XDR form separately, which indicates the second representation is meant to be preserved rather than silently discarded.
