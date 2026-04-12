# H004: Raw transaction exports omit P23+ fee-refund changes from `tx_fee_meta`

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For protocol 23+ Soroban fee-bump transactions, the exported raw fee-change payload should include the refund balance changes that occur in `PostTxApplyFeeChanges`, so downstream systems can reconstruct the full fee lifecycle from the exported XDR blobs. A transaction whose refundable resource fee is partially returned should not export a `tx_fee_meta` blob that omits the refund leg entirely.

## Mechanism

Both `TransformTransaction()` and `TransformLedgerTransaction()` serialize `transaction.FeeChanges` into `tx_fee_meta`, but post-protocol-23 refund changes moved into `transaction.PostTxApplyFeeChanges` and are not part of `UnsafeMeta`. The repository already acknowledges this split when computing `ResourceFeeRefund`, yet the raw export paths still drop those post-apply changes, leaving a plausible-but-incomplete fee metadata blob.

## Trigger

Export any protocol-23-or-later Soroban fee-bump transaction with a non-empty refundable fee refund, such as a transaction whose `PostTxApplyFeeChanges` contains the fee-account credit back after execution. Compare the exported `tx_fee_meta` with the actual `LedgerTransaction` contents: the refund change will be absent from the exported blob.

## Target Code

- `internal/transform/transaction.go:64-67` — serializes only `transaction.FeeChanges` into `TransactionOutput.TxFeeMeta`
- `internal/transform/transaction.go:195-200` — comments that P23+ refund changes moved to `PostTxApplyFeeChanges`
- `internal/transform/ledger_transaction.go:27-35` — serializes only `transaction.FeeChanges` into `LedgerTransactionOutput.TxFeeMeta`
- `internal/transform/ledger_transaction.go:47-54` — returns no field carrying `PostTxApplyFeeChanges`
- `internal/transform/transaction_test.go:178-216` — existing P23+ fixture expects a non-zero `ResourceFeeRefund`
- `internal/transform/transaction_test.go:599-600` — same fixture includes non-empty `PostTxApplyFeeChanges`

## Evidence

The production transaction transform already has special-case logic proving that `PostTxApplyFeeChanges` carries real fee-account refund data on modern ledgers. Despite that, neither raw export schema has any place to preserve those changes, so the exported XDR payload is missing part of the actual ledger transaction state.

## Anti-Evidence

Downstream consumers that only need the pre-apply fee debit or that trust the derived scalar refund columns may not notice. But consumers that treat `tx_fee_meta` as the authoritative raw fee-change blob will silently miss the P23+ refund leg.
