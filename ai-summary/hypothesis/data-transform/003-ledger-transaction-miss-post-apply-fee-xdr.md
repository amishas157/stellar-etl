# H003: `ledger_transaction` drops P23+ `PostTxApplyFeeChanges` from all raw blobs

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a protocol-23+ Soroban transaction with a fee refund, the raw
`ledger_transaction` export should preserve every fee-account mutation needed to
replay the transaction's final fee lifecycle. A consumer decoding the row's raw
XDR should be able to recover the post-apply refund changes as well as the initial
fee debit.

## Mechanism

`TransformLedgerTransaction()` base64-encodes `transaction.UnsafeMeta` into
`tx_meta` and `transaction.FeeChanges` into `tx_fee_meta`, but never serializes
`transaction.PostTxApplyFeeChanges`. Because protocol 23 moved refund credits out
of transaction meta and into that second slice, the raw ledger-transaction table
now loses refund-era fee XDR entirely on P23+ ledgers.

## Trigger

Run `export_ledger_transaction` on a protocol-23+ Soroban ledger where a
transaction receives a non-zero resource-fee refund. Decode the row's `tx_meta`
and `tx_fee_meta`: the refund credit will not appear in either blob because
`PostTxApplyFeeChanges` is not exported anywhere.

## Target Code

- `internal/transform/ledger_transaction.go:TransformLedgerTransaction:17-35` — serializes `tx_meta` and `tx_fee_meta` but never touches `PostTxApplyFeeChanges`
- `internal/transform/schema.go:LedgerTransactionOutput:86-94` — `LedgerTransactionOutput` has no field for post-apply fee-change XDR
- `internal/transform/transaction.go:TransformTransaction:195-202` — sibling transaction transform documents that P23+ refunds moved to `PostTxApplyFeeChanges`

## Evidence

The raw ledger-transaction exporter is even more direct than the history
transaction path: it emits only `UnsafeMeta`, `FeeChanges`, and `LedgerHeader`
history. Once P23 moved fee refunds into `PostTxApplyFeeChanges`, this table lost
the only raw blob that could preserve those changes, and there is no alternate
column carrying them.

## Anti-Evidence

As with `history_transactions`, the existing `tx_fee_meta` field is intentionally
the pre-apply debit blob, so the corruption is an omission rather than a bad
reinterpretation of that field. The right fix is probably to add a new raw column,
not to redefine `tx_fee_meta`.
