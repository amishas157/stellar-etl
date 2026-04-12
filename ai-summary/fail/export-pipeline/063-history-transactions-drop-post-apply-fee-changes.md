# H001: `history_transactions` drops protocol-23+ post-apply fee refund XDR

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: raw-blob fidelity / fee reconciliation
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For protocol-23+ Soroban transactions that populate `PostTxApplyFeeChanges`, the `history_transactions` row should preserve the refund-leg XDR somewhere in its exported raw blobs so downstream consumers can recompute the same fee lifecycle the ETL already reports numerically. In the repository's protocol-25 fee-bump fixture, a downstream decoder should be able to recover the 10,705-stroop refund from the exported raw fields, not just from the derived `resource_fee_refund` scalar.

## Mechanism

`TransformTransaction()` still serializes only `transaction.UnsafeMeta` into `tx_meta` and only `transaction.FeeChanges` into `tx_fee_meta`, but its own refund math now explicitly appends `transaction.PostTxApplyFeeChanges` for protocol 23+ rows. That means the numeric exports (`resource_fee_refund=10705`, `fee_charged=27828`) are derived from ledger changes that never appear in any exported raw blob, so raw-XDR consumers silently lose the refund leg and can only reconstruct the pre-refund charge.

## Trigger

Export any protocol-23+ Soroban transaction with refundable resource fees, such as the protocol-25 fee-bump fixture in `transaction_test.go` whose `PostTxApplyFeeChanges` refunds 10,705 stroops to the fee account. The emitted `tx_meta` blob will contain V4 meta without the refund changes, and `tx_fee_meta` will contain only the initial fee deduction.

## Target Code

- `cmd/export_transactions.go:25-66` — production `history_transactions` path always emits `TransformTransaction()` rows
- `internal/transform/transaction.go:49-67` — raw transaction blobs marshal only `Envelope`, `Result.Result`, `UnsafeMeta`, and `FeeChanges`
- `internal/transform/transaction.go:177-202` — protocol-23+ refund math explicitly appends `transaction.PostTxApplyFeeChanges`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction.go:21-27` — SDK carries `PostTxApplyFeeChanges` as a separate low-level field
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction.go:638-642` — upstream refund logic also appends `PostTxApplyFeeChanges`
- `internal/transform/transaction_test.go:178-217` — expected protocol-25 export already reports `resource_fee_refund=10705`
- `internal/transform/transaction_test.go:599-625` — fixture contains non-empty `PostTxApplyFeeChanges`

## Evidence

The code has already acknowledged the protocol change in comments and logic: `PostTxApplyFeeChanges` is now part of the authoritative refund computation, but the raw export surface was never extended to carry it. The test fixture proves this is not theoretical: the same row exports a non-zero refund while its asserted `TxMeta` and `TxFeeMeta` blobs contain only the pre-refund structures.

## Anti-Evidence

`tx_fee_meta` has historically represented the initial fee-deduction slice rather than a complete fee lifecycle, so the omission is not that `tx_fee_meta` itself changed semantics. The stronger claim is narrower: after protocol 23, `history_transactions` no longer exports any raw blob that preserves the refund-leg XDR that pre-23 consumers could still recover from `tx_meta`.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of data-transform/023-ledger-transaction-drops-post-tx-apply-fee-xdr and data-integrity/015-ledger-transaction-drops-post-apply-fee-changes
**Failed At**: reviewer

### Trace Summary

Traced the raw blob serialization in `TransformTransaction()` (lines 49-67) and confirmed that `tx_meta` marshals `UnsafeMeta` and `tx_fee_meta` marshals `FeeChanges`, while `PostTxApplyFeeChanges` (a separate field on `LedgerTransaction`) is not serialized into any raw blob. For V4 meta (P23+), the refund leg has moved out of `UnsafeMeta.V4.TxChangesAfter` to `LedgerTransaction.PostTxApplyFeeChanges`, so `tx_meta` no longer contains it. The derived numeric fields (`resource_fee_refund`, `fee_charged`) ARE correctly computed using `PostTxApplyFeeChanges` at lines 198-202.

### Code Paths Examined

- `internal/transform/transaction.go:59-61` — `xdr.MarshalBase64(transaction.UnsafeMeta)` serializes V4 meta without PostTxApplyFeeChanges
- `internal/transform/transaction.go:64-66` — `xdr.MarshalBase64(transaction.FeeChanges)` serializes pre-tx fee deduction only
- `internal/transform/transaction.go:198-202` — V4 path correctly appends `PostTxApplyFeeChanges` for refund math
- `internal/transform/schema.go:41-84` — `TransactionOutput` has `TxMeta` and `TxFeeMeta` but no field for PostTxApplyFeeChanges
- `internal/transform/ledger_transaction.go:27-35` — `TransformLedgerTransaction()` has identical omission
- `go-stellar-sdk/ingest/ledger_transaction.go:16-27` — SDK struct: `FeeChanges`, `UnsafeMeta`, and `PostTxApplyFeeChanges` are three separate fields

### Why It Failed

This is a duplicate of the already-confirmed success findings `data-transform/023-ledger-transaction-drops-post-tx-apply-fee-xdr` and `data-integrity/015-ledger-transaction-drops-post-apply-fee-changes`. Both success entries document the identical root cause — `PostTxApplyFeeChanges` not being serialized into any raw blob field — and explicitly reference `TransformTransaction():195-200` as sibling evidence. The mechanism, root cause, and fix are identical across `TransformTransaction()` and `TransformLedgerTransaction()`: both need a new field or expanded serialization for `PostTxApplyFeeChanges`. Any fix for the already-confirmed finding would naturally be applied to both functions simultaneously, as they share the same schema gap.

### Lesson Learned

When a finding has been confirmed for one export function (e.g., `TransformLedgerTransaction`), check whether the success entry already references sibling functions as related evidence before filing a separate hypothesis for each sibling. The existing success entries for 023/015 already noted `TransformTransaction()` as affected code, making this a re-discovery rather than a novel finding.
