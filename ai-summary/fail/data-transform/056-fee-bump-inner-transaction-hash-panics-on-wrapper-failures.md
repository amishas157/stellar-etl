# H003: failed fee-bump wrappers can panic `TransformTransaction()`

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Medium
**Impact**: data loss or transform crash on specific fee-bump failures
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`TransformTransaction()` should export a `history_transactions` row for every fee-bump envelope, including transactions rejected at the outer fee-bump wrapper layer. For wrapper-level failures such as `tx_bad_auth`, `tx_insufficient_balance`, or `tx_insufficient_fee`, the exporter should still return a row and either derive `inner_transaction_hash` from the inner envelope or leave that field empty in a controlled way instead of panicking.

## Mechanism

The fee-bump branch unconditionally calls `transaction.Result.InnerHash()` whenever `transaction.Envelope.IsFeeBump()` is true. Upstream XDR only exposes an `InnerResultPair` arm for `txFeeBumpInnerSuccess` and `txFeeBumpInnerFailed`; wrapper-level result codes use void arms, and `InnerHash()` calls `MustInnerResultPair()`, which panics when that arm is absent. A fee-bump transaction that fails before the inner transaction runs therefore crashes the transform instead of producing an export row.

## Trigger

Export a ledger containing a fee-bump transaction that fails at the outer wrapper layer, for example because the fee source has bad auth, insufficient balance, or insufficient fee. When `TransformTransaction()` reaches the fee-bump block, `transaction.Result.InnerHash()` panics because the result code is not `TxFeeBumpInnerSuccess` or `TxFeeBumpInnerFailed`.

## Target Code

- `internal/transform/transaction.go:284-300` — fee-bump branch always calls `transaction.Result.InnerHash()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/transaction_result.go:30-40` — `InnerHash()` delegates to `MustInnerResultPair()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:47707-47821` — only `TxFeeBumpInnerSuccess` and `TxFeeBumpInnerFailed` carry `InnerResultPair`; other outer failure codes do not

## Evidence

The transform tests only exercise fee-bump rows whose result code is `TransactionResultCodeTxFeeBumpInnerSuccess`, so the unconditional `InnerHash()` call is never stressed against outer-wrapper failures. The generated XDR union makes the failure mode explicit: most outer transaction result codes map to a void arm, and `MustInnerResultPair()` panics if the arm is unset.

## Anti-Evidence

If most observed fee-bump transactions either succeed or fail with `TxFeeBumpInnerFailed`, the panic stays hidden because those two result codes do carry `InnerResultPair`. The repository also mirrors the same `InnerHash()` pattern from upstream transaction processors, so reviewer confirmation is needed on whether the broader pipeline already filters wrapper-level failures before this transform runs.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the fee-bump branch in `TransformTransaction()` at `transaction.go:284-300`, confirmed the unconditional `InnerHash()` call, and verified that `MustInnerResultPair()` panics for non-fee-bump result codes. However, the Stellar protocol guarantees that fee-bump transactions in closed ledgers ALWAYS have result code `TxFeeBumpInnerSuccess` or `TxFeeBumpInnerFailed`. Wrapper-level failures (bad auth, insufficient balance, insufficient fee) are caught during validator-level validation and prevent the transaction from being included in the ledger at all. The trigger scenario is impossible under the protocol.

### Code Paths Examined

- `internal/transform/transaction.go:284-293` — fee-bump branch unconditionally calls `transaction.Result.InnerHash()`; pattern is identical to upstream Horizon
- `go-stellar-sdk/xdr/transaction_result.go:38-40` — `InnerHash()` delegates to `MustInnerResultPair()` which panics on void arms
- `go-stellar-sdk/xdr/xdr_generated.go:47705-47740` — `ArmForSwitch` confirms only `TxFeeBumpInnerSuccess` and `TxFeeBumpInnerFailed` carry `InnerResultPair`; all other codes map to void arms
- `go-stellar-sdk/ingest/ledger_transaction.go:729-737` — the SDK's own `InnerTransactionHash()` uses the exact same unconditional `InnerHash()` pattern
- `stellar/go/services/horizon/.../transaction_batch_insert_builder.go:211-213` — Horizon's production transaction processor uses the exact same unconditional pattern

### Why It Failed

The hypothesis assumes that a fee-bump transaction can appear in a closed ledger with a wrapper-level failure result code (e.g., `TxBadAuth`, `TxInsufficientBalance`, `TxInsufficientFee`). This is incorrect. The Stellar protocol validates the outer fee-bump wrapper during transaction set trimming — if the fee source has bad auth, insufficient balance, or insufficient fee, the transaction is rejected and never included in the ledger. By the time a fee-bump transaction appears in a `LedgerTransaction` (read from history archives), its result code is guaranteed to be either `TxFeeBumpInnerSuccess` or `TxFeeBumpInnerFailed`, both of which carry the `InnerResultPair` arm. The unconditional `InnerHash()` call is safe by protocol invariant.

This is corroborated by the fact that both the upstream Horizon transaction processor and the SDK's own `LedgerTransaction.InnerTransactionHash()` method use the identical unconditional pattern — neither guards against non-fee-bump result codes, and both have been running in production for years without panics.

### Lesson Learned

Fee-bump wrapper-level failures are caught during stellar-core's transaction set validation phase and never make it into closed ledgers. The only result codes possible for fee-bump transactions in history archives are `TxFeeBumpInnerSuccess` and `TxFeeBumpInnerFailed`. Before hypothesizing a panic from a `Must*()` call, verify whether the protocol guarantees that the union arm is always populated in the data the ETL actually processes. The upstream Horizon codebase using the same "unguarded" pattern is strong evidence of a protocol-level invariant.
