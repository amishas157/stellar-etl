# H003: Contract-data Parquet drops `ledger_key_hash_base_64`

**Date**: 2026-04-15
**Subsystem**: external-io
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --export-contract-data --write-parquet` writes a contract-data row, the Parquet output should preserve the same reversible base64-encoded ledger key that the JSON/output struct exposes as `ledger_key_hash_base_64`. Consumers should not lose the original ledger-key encoding just because they read Parquet instead of JSON.

## Mechanism

`TransformContractData()` computes `LedgerKeyHashBase64` from `ledgerEntry.LedgerKey()` and stores it on `ContractDataOutput`, but `ContractDataOutputParquet` omits the column and `ContractDataOutput.ToParquet()` never copies it. The Parquet export therefore keeps only the irreversible `ledger_key_hash` digest and silently discards the reversible base64 ledger-key encoding for every contract-data row.

## Trigger

Run `export_ledger_entry_changes --export-contract-data --write-parquet` over any ledger range containing a non-nonce `LedgerEntryTypeContractData` change. The JSON row will include a populated `ledger_key_hash_base_64`, but the Parquet row schema has no matching column and cannot retain it.

## Target Code

- `internal/transform/contract_data.go:TransformContractData:65-78,135-156` — computes `ledgerKeyHashBase64` from `ledgerEntry.LedgerKey()` and stores it on `ContractDataOutput`
- `internal/transform/schema.go:ContractDataOutput:515-537` — JSON/output schema includes `ledger_key_hash_base_64`
- `internal/transform/schema_parquet.go:ContractDataOutputParquet:260-281` — Parquet schema omits the base64 field
- `internal/transform/parquet_converter.go:ContractDataOutput.ToParquet:292-313` — converter copies `LedgerKeyHash` but drops `LedgerKeyHashBase64`
- `cmd/export_ledger_entry_changes.go:exportTransformedData:344-347,370-372` — production path selects `ContractDataOutputParquet` and writes only the reduced Parquet struct

## Evidence

The transform does real work to derive `LedgerKeyHashBase64`: it calls `ledgerEntry.LedgerKey()`, marshals that key to base64, and stores the result alongside the irreversible hash. Because the Parquet row type excludes only this field while preserving the rest of the contract-data row, the omission is a concrete JSON/Parquet mismatch rather than a wholesale unsupported export.

## Anti-Evidence

Parquet consumers still receive `ledger_key_hash`, `key`, `key_decoded`, `val`, and `contract_data_xdr`, so the row does not look obviously broken. But the exporter explicitly models `ledger_key_hash_base_64` as part of the output contract, and dropping it removes the only reversible ledger-key encoding from Parquet consumers.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/success/data-transform/006-contract-data-parquet-drops-ledger-key-hash-base64.md.gh-published`
**Failed At**: reviewer

### Trace Summary

The hypothesis correctly identifies that `ContractDataOutputParquet` (schema_parquet.go:260-281) omits the `LedgerKeyHashBase64` field present in `ContractDataOutput` (schema.go:536), and that `ToParquet()` (parquet_converter.go:292-313) therefore cannot carry the value. The bug is real and the code path is confirmed. However, this exact finding has already been confirmed and published under the `data-transform` subsystem.

### Code Paths Examined

- `internal/transform/contract_data.go:TransformContractData:65-78` — confirmed `ledgerKeyHashBase64` computed via `xdr.MarshalBase64(ledgerKey)` and stored on `ContractDataOutput.LedgerKeyHashBase64`
- `internal/transform/schema.go:ContractDataOutput:536` — confirmed `LedgerKeyHashBase64 string` field with json tag `ledger_key_hash_base_64`
- `internal/transform/schema_parquet.go:ContractDataOutputParquet:260-281` — confirmed no `LedgerKeyHashBase64` field; schema ends at `ContractDataXDR`
- `internal/transform/parquet_converter.go:ContractDataOutput.ToParquet:292-313` — confirmed no `LedgerKeyHashBase64` assignment; last field copied is `ContractDataXDR`

### Why It Failed

This is an exact duplicate of `ai-summary/success/data-transform/006-contract-data-parquet-drops-ledger-key-hash-base64.md.gh-published`, which identifies the same root cause (missing `LedgerKeyHashBase64` in `ContractDataOutputParquet` and `ToParquet()`), same affected code paths, and same impact. The only difference is the subsystem label (`external-io` vs `data-transform`).

### Lesson Learned

The "Parquet drops `ledger_key_hash_base_64`" pattern for contract-data was already confirmed via the `data-transform` subsystem (006). Both sibling entity types (contract-code via success/external-io/022 and contract-data via success/data-transform/006) are already documented. Always check success directories across ALL subsystems before filing.
