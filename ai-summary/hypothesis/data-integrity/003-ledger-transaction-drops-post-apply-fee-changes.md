# H003: `export_ledger_transaction` drops P23+ refund changes from raw fee metadata

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For protocol-23+ Soroban transactions that refund part of the resource fee, the raw ledger-transaction export should preserve all fee-related balance changes needed to reconstruct the actual charged fee and refund. A downstream decoder of `tx_fee_meta` should see the refund leg, or the export should provide a separate field for `PostTxApplyFeeChanges`.

## Mechanism

`TransformLedgerTransaction()` serializes only `transaction.FeeChanges` into `tx_fee_meta` and never exports `transaction.PostTxApplyFeeChanges` anywhere. The same repository's higher-level `TransformTransaction()` and the upstream ingest comments both acknowledge that, from P23 onward, refund balance changes moved out of `TxChangesAfter` into `PostTxApplyFeeChanges`, so the raw `ledger_transaction` row silently omits part of the real fee flow exactly when downstream fee reconciliation needs it most.

## Trigger

Run `export_ledger_transaction` on a protocol-23+ fee-bump Soroban transaction with a non-zero resource-fee refund in `PostTxApplyFeeChanges`.

## Target Code

- `internal/transform/ledger_transaction.go:27-39` — serializes `UnsafeMeta` and `FeeChanges`, but not `PostTxApplyFeeChanges`
- `internal/transform/ledger_transaction.go:47-55` — output struct has no field for post-apply fee changes
- `internal/transform/transaction.go:195-202` — repository already documents that P23+ refunds moved into `PostTxApplyFeeChanges`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/ingest/ledger_transaction.go:21-27` — `LedgerTransaction` carries `FeeChanges`, `UnsafeMeta`, and `PostTxApplyFeeChanges` separately
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/ingest/ledger_transaction.go:639-642` — upstream ingest explicitly appends `PostTxApplyFeeChanges` when computing P23+ refunds

## Evidence

The repository already fixed the higher-level transaction export to include `PostTxApplyFeeChanges` when computing `resource_fee_refund`, which is strong evidence that those changes are semantically necessary and not optional metadata. The raw ledger-transaction export still emits only `FeeChanges`, so a consumer decoding `tx_fee_meta` gets an incomplete fee-change set for the exact class of transactions where refunds exist.

## Anti-Evidence

Some downstream users may rely on the separately exported `tx_meta`/`tx_ledger_history` fields instead of `tx_fee_meta`, so not every consumer is affected. This hypothesis assumes the intent of `tx_fee_meta` is to preserve the full fee-change stream rather than just the pre-apply subset.
