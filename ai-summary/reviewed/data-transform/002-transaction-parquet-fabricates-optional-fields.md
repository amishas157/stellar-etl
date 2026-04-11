# H002: Transaction parquet fabricates absent optional fields

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For transactions that do not carry optional muxed-account or fee-bump metadata, the Parquet export should preserve those columns as null / absent, matching the JSON output. In particular, classic non-fee-bump transactions should not acquire synthetic values for `account_muxed`, `fee_account`, `fee_account_muxed`, `inner_transaction_hash`, or `new_max_fee`.

## Mechanism

`TransformTransaction()` only populates these fields in the muxed-account and fee-bump branches, and the JSON schema marks them optional with `omitempty`. The Parquet schema instead defines them as required `string` / `int64` columns, and `ToParquet()` copies zero values directly, so classic rows are exported with fabricated `""` and `0` values that look like real data rather than missing data.

## Trigger

Export any classic transaction whose source account is not muxed and whose envelope is not fee-bump. The JSON row omits these fields, while the Parquet row serializes empty strings and `new_max_fee = 0`.

## Target Code

- `internal/transform/transaction.go:273-301` — only sets muxed-account and fee-bump fields inside conditional branches
- `internal/transform/schema.go:45,60-63` — JSON schema declares these fields optional with `omitempty`
- `internal/transform/schema_parquet.go:36,51-55` — Parquet schema forces concrete string / int64 columns
- `internal/transform/parquet_converter.go:64,79-83` — converter writes zero values into Parquet rows

## Evidence

The transaction test fixture's first classic transaction leaves all fee-bump-only fields unset, which matches the JSON contract. The Parquet path has no nullable wrapper for those same fields, so absent metadata is converted into concrete empty values during serialization.

## Anti-Evidence

An empty string is not a valid Stellar account or transaction hash, so downstream consumers can sometimes infer that the value is synthetic. But that still changes the output contract from "missing" to "present with a value," which breaks null-sensitive analytics and is the same class of corruption already proven elsewhere in the repository for other optional columns.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated. Success finding 003 covers `min_account_sequence` (null.Int fields); 010 covers `sponsor` (null.String fields); 011 covers effects `address_muxed` (null.String). This hypothesis covers a distinct set of transaction fields using plain string/int64 with `omitempty`.

### Trace Summary

`TransformTransaction()` in `transaction.go` constructs a `TransactionOutput` struct with all fields initialized to Go zero values. The fee-bump/muxed fields (`AccountMuxed`, `FeeAccount`, `FeeAccountMuxed`, `InnerTransactionHash`, `NewMaxFee`) are only populated inside conditional branches (lines 274-301) that check for muxed source accounts and fee-bump envelopes. For classic transactions, these remain at zero values ("" and 0). The JSON schema uses `omitempty` tags to correctly omit these, but `ToParquet()` copies the zero values directly into required Parquet columns, fabricating present-but-empty data.

### Code Paths Examined

- `internal/transform/transaction.go:273-281` — `AccountMuxed` is only set when `sourceAccount.Type == xdr.CryptoKeyTypeKeyTypeMuxedEd25519`; otherwise stays ""
- `internal/transform/transaction.go:284-301` — `FeeAccount`, `FeeAccountMuxed`, `InnerTransactionHash`, `NewMaxFee` are only set inside `transaction.Envelope.IsFeeBump()` block; otherwise stay at zero values
- `internal/transform/schema.go:45,60-63` — JSON tags `account_muxed,omitempty`, `fee_account,omitempty`, `fee_account_muxed,omitempty`, `inner_transaction_hash,omitempty`, `new_max_fee,omitempty` correctly omit zero values
- `internal/transform/schema_parquet.go:36,51-54` — Parquet schema defines all five as required (non-optional) columns: `string` for text fields, `int64` for `NewMaxFee`
- `internal/transform/parquet_converter.go:64,79-82` — `ToParquet()` directly copies: `to.AccountMuxed`, `to.FeeAccount`, `to.FeeAccountMuxed`, `to.InnerTransactionHash`, `int64(to.NewMaxFee)` — no null/optional handling

### Findings

The bug is confirmed and follows the exact same pattern as three already-confirmed findings (003, 010, 011) in this codebase. For classic (non-fee-bump, non-muxed) transactions:

1. **JSON output**: Fields are absent (omitted by `omitempty`) — correct behavior
2. **Parquet output**: Fields contain `""` (strings) or `0` (NewMaxFee) — fabricated data

The five affected columns are:
- `account_muxed`: "" instead of null (affects ~99%+ of transactions since muxed accounts are rare)
- `fee_account`: "" instead of null (affects all non-fee-bump transactions)
- `fee_account_muxed`: "" instead of null (affects all non-fee-bump transactions and fee-bumps with non-muxed fee accounts)
- `inner_transaction_hash`: "" instead of null (affects all non-fee-bump transactions)
- `new_max_fee`: 0 instead of null (affects all non-fee-bump transactions)

This breaks null-sensitive analytics. For example, `SELECT COUNT(*) FROM transactions WHERE fee_account IS NOT NULL` would count ALL transactions instead of only fee-bump transactions, because `""` is not NULL.

### PoC Guidance

- **Test file**: `internal/transform/data_integrity_poc_test.go` (append to existing file if it exists, otherwise create)
- **Setup**: Use `makeTransactionTestInput()` to get a classic (non-fee-bump) transaction fixture. Transform it with `TransformTransaction()`.
- **Steps**: (1) Confirm the JSON-layer output has zero-value fields that would be omitted by `omitempty`. (2) Call `ToParquet()` on the output. (3) Cast the result to `TransactionOutputParquet`.
- **Assertion**: Assert that `parquet.FeeAccount == ""` and `parquet.NewMaxFee == 0` — demonstrating that absent optional fields are fabricated as concrete values in Parquet output. Contrast this with the JSON behavior where these fields would be omitted entirely.
