# H002: Fee-bump transactions export wrapper result codes instead of the inner transaction result

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: fee-bump rows split semantically equivalent transaction outcomes into different result-code buckets
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For fee-bump rows in `history_transactions`, `transaction_result_code` should preserve the wrapped inner transaction's actual result code so the column remains comparable with non-fee-bump transactions. A fee-bump whose inner transaction succeeded should therefore export `TransactionResultCodeTxSuccess`, and one whose inner transaction failed for `tx_insufficient_balance` should export that same inner failure code, with outer fee-bump wrapper state represented separately if needed.

## Mechanism

`TransformTransaction()` always serializes `transaction.Result.Result.Result.Code.String()`, which for fee-bump envelopes is the outer wrapper union arm (`TxFeeBumpInnerSuccess` / `TxFeeBumpInnerFailed`). The XDR result union simultaneously carries the inner transaction result pair, but the transform never descends into it when populating `transaction_result_code`. That makes fee-bump rows use a different result-code vocabulary than classic rows even when the inner transaction outcome is semantically identical.

## Trigger

1. Export a fee-bump transaction row from `history_transactions`.
2. Compare `transaction_result_code` against the inner transaction result stored inside `InnerResultPair`.
3. The ETL will emit the outer wrapper status (`TxFeeBumpInnerSuccess` or `TxFeeBumpInnerFailed`) instead of the inner transaction's concrete result code.

## Target Code

- `internal/transform/transaction.go:225-225` — `transaction_result_code` is sourced from the outer result union
- `internal/transform/transaction.go:283-300` — fee-bump-specific fields are extracted later, but inner result code is never used
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:47691-47765` — fee-bump wrapper codes still carry `InnerResultPair`
- `internal/transform/transaction_test.go:142-142` — current unit test expects `TransactionResultCodeTxFeeBumpInnerSuccess`

## Evidence

The generated XDR union exposes `InnerResultPair` specifically for `TxFeeBumpInnerSuccess` and `TxFeeBumpInnerFailed`, so the inner transaction result is available at transform time. Nonetheless the transform records only the outer code string, and the test fixture explicitly locks in that wrapper-level value.

## Anti-Evidence

The current tests intentionally assert the wrapper code, so maintainers may consider `transaction_result_code` to mean the top-level XDR union discriminant rather than the wrapped transaction outcome. The table also exposes fee-bump-specific fields (`fee_account`, `inner_transaction_hash`, `new_max_fee`) that could be taken as evidence that wrapper-specific semantics are intentional.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (related fee-bump topics 006, 033, 048, 056 in summary.md cover different fields)
**Failed At**: reviewer

### Trace Summary

Traced the full code path from `TransformTransaction()` at `transaction.go:225` through the XDR type chain `TransactionResultPair.Result` → `TransactionResult.Result` → `TransactionResultResult.Code`. Confirmed that `.Code` is the XDR union discriminant, which for fee-bump transactions is `TxFeeBumpInnerSuccess` or `TxFeeBumpInnerFailed` — these are the protocol-defined result codes for fee-bump transactions. The XDR specification defines these as distinct enum values (1 and -13 respectively) that are structurally different from classic result codes.

### Code Paths Examined

- `internal/transform/transaction.go:225` — `outputTxResultCode := transaction.Result.Result.Result.Code.String()` faithfully serializes the XDR union discriminant
- `internal/transform/transaction.go:283-300` — fee-bump block correctly extracts `FeeAccount`, `InnerTransactionHash`, `NewMaxFee`, and outer `TxSigners`
- `internal/transform/transaction_test.go:142` — test explicitly expects `"TransactionResultCodeTxFeeBumpInnerSuccess"` with comment "//inner fee bump success"
- `go-stellar-sdk/xdr/xdr_generated.go:47691-47695` — `TransactionResultResult` struct confirms `Code` is the union discriminant, `InnerResultPair` is a separate arm
- `go-stellar-sdk/xdr/transaction_result.go:6-9` — `Successful()` method treats both `TxSuccess` and `TxFeeBumpInnerSuccess` as success, confirming these are distinct protocol-level codes

### Why It Failed

This describes **working-as-designed behavior**. The `transaction_result_code` field correctly exports the XDR union discriminant, which for fee-bump transactions IS `TxFeeBumpInnerSuccess` / `TxFeeBumpInnerFailed` per the Stellar protocol specification. These are not "wrapper codes hiding the real result" — they ARE the result codes defined by the protocol for fee-bump transactions. The hypothesis's "Expected Behavior" section incorrectly assumes the column should normalize fee-bump results to classic result codes, but:

1. The XDR protocol defines these as the correct top-level result codes for fee-bump transactions
2. The tests explicitly and intentionally assert this behavior
3. The ETL already exports fee-bump-specific fields (`fee_account`, `inner_transaction_hash`, `new_max_fee`) enabling downstream consumers to identify and handle fee-bump transactions
4. This matches the established fee-bump field-split convention documented in summary.md (lesson 12): the ETL mirrors Horizon's convention for fee-bump transactions

### Lesson Learned

For fee-bump transactions, all exported fields follow a consistent convention: they represent the top-level XDR values, not unwrapped inner values. Result codes, signers, and fee fields all follow this pattern. The inner transaction's details are accessible through the dedicated fee-bump fields. Before hypothesizing that a field should "unwrap" an inner value, verify whether the existing convention is to export the outer XDR discriminant directly.
