# H002: `history_transactions` drops P23+ `PostTxApplyFeeChanges` from all raw blobs

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a protocol-23+ Soroban transaction with a non-zero fee refund, the exported
`history_transactions` row should preserve the full fee lifecycle in its raw XDR
columns. Downstream consumers should be able to decode the row's blobs and recover
both the pre-apply fee debit and the post-apply refund credit.

## Mechanism

`TransformTransaction()` serializes `tx_meta` from `transaction.UnsafeMeta` and
`tx_fee_meta` from `transaction.FeeChanges`, but the same function documents that
P23+ refund balance changes moved into `transaction.PostTxApplyFeeChanges`. The
numeric refund columns append that slice when computing `resource_fee_refund`, yet
`TransactionOutput` has no field that stores the underlying post-apply XDR, so the
row can report a refund numerically while exposing no raw blob that contains it.

## Trigger

Run `export_transactions` on a protocol-23+ Soroban ledger containing a
transaction with `resource_fee_refund > 0`. Decode `tx_meta` and `tx_fee_meta`:
neither blob will include the refund changes even though the same row exports a
non-zero `resource_fee_refund`.

## Target Code

- `internal/transform/transaction.go:TransformTransaction:59-65` — serializes only `transaction.UnsafeMeta` and `transaction.FeeChanges`
- `internal/transform/transaction.go:TransformTransaction:195-202` — explicitly documents that P23+ refunds moved to `transaction.PostTxApplyFeeChanges`
- `internal/transform/schema.go:TransactionOutput:50-53` — `history_transactions` exposes `tx_envelope`, `tx_result`, `tx_meta`, and `tx_fee_meta`, but no post-apply fee-change blob
- `internal/transform/schema_parquet.go:TransactionOutputParquet:41-44` — Parquet mirrors the same raw-blob set with no `PostTxApplyFeeChanges` column

## Evidence

The code comment at lines 195-197 is explicit: on P23+, the refund balance change
no longer lives inside transaction meta and must be read from
`PostTxApplyFeeChanges`. That slice is already trusted for the numeric
`resource_fee_refund` calculation, proving the row knows about the refund while
still omitting any raw XDR column that preserves it.

## Anti-Evidence

`tx_fee_meta` itself is intentionally the pre-apply fee-debit blob, so this is
not a request to repurpose that existing field. The likely fix is additive: expose
the post-apply fee-change slice in a new raw column rather than changing current
blob semantics.
