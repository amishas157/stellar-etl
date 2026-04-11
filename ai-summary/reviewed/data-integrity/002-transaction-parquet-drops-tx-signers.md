# H002: Transaction Parquet export drops populated `tx_signers`

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_transactions --write-parquet` is used, the Parquet row should preserve the same signer list that the JSON row exposes in `tx_signers`. Transactions with one or more envelope signatures should therefore yield a populated repeated/string column rather than silently losing signer provenance.

## Mechanism

`TransformTransaction()` populates `TxSigners`, but `TransactionOutputParquet` has no `tx_signers` field and `TransactionOutput.ToParquet()` never maps one. The export command still writes Parquet for every transformed transaction, so the signer list disappears entirely on the Parquet path even though the JSON export and in-memory struct contain it.

## Trigger

Run `export_transactions --write-parquet` on any ledger containing a signed transaction. The JSON output will include `tx_signers`, while the generated Parquet schema/file will have no `tx_signers` column at all.

## Target Code

- `internal/transform/transaction.go:227-301` — derives and stores `TxSigners`
- `internal/transform/schema.go:42-84` — declares `tx_signers` on `TransactionOutput`
- `internal/transform/schema_parquet.go:31-74` — defines `TransactionOutputParquet` without a `tx_signers` field
- `internal/transform/parquet_converter.go:59-102` — converts transactions to Parquet without mapping `TxSigners`
- `cmd/export_transactions.go:51-65` — appends transformed rows and writes the Parquet file

## Evidence

The transform test fixtures already expect non-empty `TxSigners` slices for normal, fee-bump, and Soroban transactions. The Parquet schema includes nearby repeated fields such as `extra_signers`, which shows the exporter already supports list-valued transaction columns and that `tx_signers` is not intentionally impossible to serialize.

## Anti-Evidence

JSON exports remain correct because `ExportEntry()` serializes the original `TransactionOutput`. Transactions with no signatures would not make the omission visible, but Stellar transaction envelopes normally contain at least one signature.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformTransaction()` (transaction.go:227-270) extracts envelope signatures via `getTxSigners()` and stores them in `TransactionOutput.TxSigners`. For fee-bump transactions, the signers are overwritten at line 295-300 with the fee-bump envelope signatures. The JSON output struct (`TransactionOutput`, schema.go:83) declares `TxSigners []string` with a `json:"tx_signers"` tag, and tests confirm non-empty values for all transaction types. However, `TransactionOutputParquet` (schema_parquet.go:31-74) has no corresponding field, and `ToParquet()` (parquet_converter.go:59-102) never maps it — the field is silently dropped during Parquet conversion.

### Code Paths Examined

- `internal/transform/schema.go:83` — `TxSigners []string` declared on `TransactionOutput`
- `internal/transform/schema_parquet.go:31-74` — `TransactionOutputParquet` struct ends at line 74 with `RentFeeCharged`; no `TxSigners` field exists
- `internal/transform/parquet_converter.go:59-102` — `ToParquet()` maps all fields except `TxSigners`; conversion ends at `RentFeeCharged` (line 101)
- `internal/transform/transaction.go:227-270` — `getTxSigners()` called and result stored in `TxSigners` field
- `internal/transform/transaction.go:295-300` — Fee-bump path re-extracts and overwrites `TxSigners`
- `internal/transform/transaction.go:349-361` — `getTxSigners()` encodes each `DecoratedSignature` as a strkey account ID
- `internal/transform/transaction_test.go:112,143,176,216` — Tests expect non-empty `TxSigners` slices for all transaction types

### Findings

The bug is a straightforward field omission in the Parquet schema and converter. Every other field present in `TransactionOutput` has a corresponding field in `TransactionOutputParquet` and a mapping line in `ToParquet()`. `TxSigners` is the sole exception. The existing `ExtraSigners` field (schema_parquet.go:59) demonstrates the pattern for list-of-string columns in Parquet using `type=MAP, convertedtype=LIST, valuetype=BYTE_ARRAY`, confirming this is not a Parquet serialization limitation. The fix requires adding a `TxSigners` field to `TransactionOutputParquet` with appropriate parquet tags, and adding the mapping line `TxSigners: to.TxSigners` in `ToParquet()`.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (or `internal/transform/transaction_test.go` if no dedicated parquet converter test exists)
- **Setup**: Use an existing test transaction fixture that produces a non-empty `TxSigners` slice (e.g., the standard or fee-bump test cases from `transaction_test.go`)
- **Steps**: Call `TransformTransaction()` on the fixture, then call `.ToParquet()` on the result. Cast the returned `interface{}` to `TransactionOutputParquet`.
- **Assertion**: Verify that the returned `TransactionOutputParquet` has a `TxSigners` field — currently it won't compile because the field doesn't exist, which directly demonstrates the data loss. Alternatively, use reflection to check that every exported field in `TransactionOutput` has a corresponding field in `TransactionOutputParquet`.
