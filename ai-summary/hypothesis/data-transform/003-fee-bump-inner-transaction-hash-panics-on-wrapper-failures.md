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
