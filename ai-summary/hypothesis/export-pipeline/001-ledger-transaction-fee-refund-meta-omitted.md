# H001: `export_ledger_transaction` drops P23+ post-apply fee refunds from `tx_fee_meta`

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: fee-refund metadata omitted
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For protocol-23+ Soroban transactions, `tx_fee_meta` should preserve every fee-related ledger-entry change needed to reconstruct the final fee debit and refund. A downstream decoder should be able to read the exported blob and recover both the initial fee charge and any `PostTxApplyFeeChanges` refund that landed after transaction execution.

## Mechanism

`ingest.LedgerTransaction` splits fee movement across two XDR slices: `FeeChanges` and `PostTxApplyFeeChanges`. `TransformLedgerTransaction()` serializes only `transaction.FeeChanges`, even though `TransformTransaction()` already documents that P23+ refunds moved into `transaction.PostTxApplyFeeChanges` and explicitly appends that slice when computing `resource_fee_refund`, so the raw `tx_fee_meta` export silently omits the refund half of the fee story.

## Trigger

Run `export_ledger_transaction` over any protocol-23+ ledger containing a Soroban transaction with a positive resource-fee refund (for example, a fee-bump Soroban transaction whose final charged resource fee is lower than the initial bid).

## Target Code

- `internal/transform/ledger_transaction.go:32-35` — marshals only `transaction.FeeChanges` into `tx_fee_meta`
- `internal/transform/transaction.go:195-202` — sibling transaction export already appends `transaction.PostTxApplyFeeChanges` for P23+ refund accounting
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/ingest/ledger_transaction.go:21-27` — refund changes live in a distinct `PostTxApplyFeeChanges` field
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/ingest/ledger_transaction.go:60-69` — upstream ingest explicitly labels that slice as fee-refund changes

## Evidence

The raw exporter never references `transaction.PostTxApplyFeeChanges`, while the normal transaction transformer has an explicit P23+ comment saying the refund balance change moved there. That means the same transaction can produce a non-zero `resource_fee_refund` in `history_transactions` while `export_ledger_transaction.tx_fee_meta` still decodes to a blob missing the refund ledger changes.

## Anti-Evidence

Classic and pre-P23 transactions keep all fee movement in older metadata layouts, so they are unaffected. If the product contract intended `tx_fee_meta` to mean only pre-apply fee debits, the current field name does not make that restriction explicit and downstream raw-metadata consumers have no alternate exported refund field.
