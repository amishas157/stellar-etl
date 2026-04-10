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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformContractData()` at `contract_data.go:65-78` marshals the ledger key to base64 via `xdr.MarshalBase64(ledgerKey)` and stores it in `ledgerKeyHashBase64`. This value is assigned to `ContractDataOutput.LedgerKeyHashBase64` at line 155. The JSON schema at `schema.go:536` declares the field with tag `json:"ledger_key_hash_base_64"`. However, `ContractDataOutputParquet` (schema_parquet.go:261-281) has no `LedgerKeyHashBase64` field — it ends at `ContractDataXDR` on line 280. The `ToParquet()` converter (parquet_converter.go:292-313) likewise stops at `ContractDataXDR` and never copies `LedgerKeyHashBase64`. This is the same class of bug as the confirmed finding in success/002 (contract event parquet dropping operation_id).

### Code Paths Examined

- `internal/transform/contract_data.go:65-78` — confirmed: computes `ledgerKeyHashBase64` from `xdr.MarshalBase64(ledgerKey)` and errors out if marshaling fails, so the value is always populated for non-nonce rows
- `internal/transform/contract_data.go:135-156` — confirmed: `LedgerKeyHashBase64: ledgerKeyHashBase64` assigned in the output struct literal
- `internal/transform/schema.go:536` — confirmed: `LedgerKeyHashBase64 string` with JSON tag `ledger_key_hash_base_64` is the last field of `ContractDataOutput`
- `internal/transform/schema_parquet.go:260-281` — confirmed: `ContractDataOutputParquet` has 20 fields ending with `ContractDataXDR`; no `LedgerKeyHashBase64` field exists
- `internal/transform/parquet_converter.go:292-313` — confirmed: `ToParquet()` maps all 20 fields but omits `LedgerKeyHashBase64`; the last assignment is `ContractDataXDR: cdo.ContractDataXDR`

### Findings

The bug is a straightforward omission: when `LedgerKeyHashBase64` was added to the JSON schema, neither the Parquet schema struct nor the Parquet converter were updated to carry the field. Every non-nonce contract-data row exported via the Parquet path silently loses its base64 ledger key encoding. The hex `LedgerKeyHash` is preserved, so one identifier survives, but the two are semantically different — the hex is a hash of the ledger key, while the base64 is a serialization of the full ledger key XDR, enabling reconstruction of the original key. Dropping it is data loss.

### PoC Guidance

- **Test file**: `internal/transform/contract_data_test.go` (append a new test)
- **Setup**: Use existing test fixtures to create a `ContractDataOutput` with a populated `LedgerKeyHashBase64` field; alternatively, construct a minimal `ContractDataOutput` struct literal with `LedgerKeyHashBase64` set to a non-empty string
- **Steps**: Call `ToParquet()` on the `ContractDataOutput` and cast the result to `ContractDataOutputParquet`
- **Assertion**: Verify that `ContractDataOutputParquet` has a `LedgerKeyHashBase64` field and that it equals the source value. Under the current code, the Parquet struct lacks the field entirely, so this test will fail to compile — demonstrating that the field is structurally absent from the Parquet schema
