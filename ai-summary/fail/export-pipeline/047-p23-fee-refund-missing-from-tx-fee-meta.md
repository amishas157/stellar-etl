# H001: P23 fee refunds are missing from exported `tx_fee_meta`

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For protocol 23+ Soroban fee-bump transactions that populate `PostTxApplyFeeChanges`, the exported `tx_fee_meta` field should encode the full fee-account lifecycle for that transaction: the initial fee deduction plus the post-apply refund. Decoding `tx_fee_meta` should therefore let downstream consumers reconstruct the same net charged fee that the exporter already reports in `resource_fee_refund`, `inclusion_fee_charged`, and the raw ledger history.

## Mechanism

Both transaction exporters serialize `transaction.FeeChanges` directly into `tx_fee_meta`, but on protocol 23+ the refund leg moved out of `FeeChanges` and into `transaction.PostTxApplyFeeChanges`. `TransformTransaction()` already knows this for numeric fee math and appends `PostTxApplyFeeChanges` when computing `resource_fee_refund`, so the raw metadata blob now disagrees with the exporter's own derived fee fields and silently overstates the charged amount when consumers trust `tx_fee_meta`.

## Trigger

Export any protocol 23+ fee-bump Soroban transaction with a refundable resource fee, such as the real mainnet-style V4-meta case embedded in `transaction_test.go` where `FeeChanges` deducts 38,533 stroops and `PostTxApplyFeeChanges` refunds 10,705 stroops. The emitted `tx_fee_meta` blob will contain only the initial deduction changes and omit the refund changes entirely.

## Target Code

- `internal/transform/transaction.go:64-67` — `tx_fee_meta` marshals only `transaction.FeeChanges`
- `internal/transform/transaction.go:195-202` — refund math for P23+ explicitly appends `transaction.PostTxApplyFeeChanges`
- `internal/transform/ledger_transaction.go:27-35` — raw ledger-transaction export repeats the same omission
- `internal/transform/transaction_test.go:178-217` — expected output includes `resource_fee_refund=10705` for a P23+ fee-bump transaction
- `internal/transform/transaction_test.go:471-625` — test fixture provides non-empty `PostTxApplyFeeChanges`

## Evidence

The code now has a split-brain model for fee metadata: numeric fee outputs were updated for protocol 23+ (`feeChanges := append(metav4.TxChangesAfter, transaction.PostTxApplyFeeChanges...)`), but `tx_fee_meta` still serializes only `transaction.FeeChanges`. The test fixture proves the trigger is real inside this repository: the same transaction carries both an initial fee deduction and a later refund, yet the asserted `TxFeeMeta` string corresponds only to the initial `FeeChanges` slice.

## Anti-Evidence

`tx_ledger_history` still contains the full ledger close meta, so a sufficiently sophisticated downstream consumer could recover the refund by reparsing a different blob. But the dedicated `tx_fee_meta` column is supposed to be the fee-specific export surface, and the exporter's own derived refund fields already treat `PostTxApplyFeeChanges` as fee metadata, so leaving it out of `tx_fee_meta` is still a concrete correctness gap.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the full fee data flow across protocol versions. `FeeChanges` is populated by the SDK from `LedgerCloseMeta.txProcessing[i].feeProcessing` and represents the **pre-transaction fee deduction only**. It has never contained refund data in any protocol version. Before P23 (V3 meta), refund data lived in `UnsafeMeta.V3.TxChangesAfter`, which was exported via `tx_meta` — not `tx_fee_meta`. In P23+ (V4 meta), refund data moved to `PostTxApplyFeeChanges` on `LedgerTransaction`, which is outside `UnsafeMeta.V4`. The hypothesis's core claim that refund data "moved out of `FeeChanges`" is factually incorrect.

### Code Paths Examined

- `internal/transform/transaction.go:64` — `xdr.MarshalBase64(transaction.FeeChanges)` serializes pre-tx fee deduction. This has always been deduction-only.
- `internal/transform/transaction.go:177-179` — `getAccountBalanceFromLedgerEntryChanges(transaction.FeeChanges, ...)` computes `initialFeeCharged` from the deduction in FeeChanges.
- `internal/transform/transaction.go:181-193` — V3 path: refund computed from `meta.TxChangesAfter` (part of `UnsafeMeta`), NOT from `FeeChanges`.
- `internal/transform/transaction.go:195-200` — V4 path: refund computed from `metav4.TxChangesAfter` + `PostTxApplyFeeChanges`, again NOT from `FeeChanges`.
- `internal/transform/ledger_transaction.go:32` — Same `xdr.MarshalBase64(transaction.FeeChanges)` pattern, consistent with transaction.go.
- `go-stellar-sdk/ingest/ledger_transaction.go:25-27` — SDK struct: `FeeChanges` and `PostTxApplyFeeChanges` are architecturally separate fields with distinct lifecycle semantics.
- `go-stellar-sdk/ingest/ledger_transaction.go:50-58` — `GetFeeChanges()` operates only on `FeeChanges`; separate `GetPostApplyFeeChanges()` (lines 62-70) handles refunds.
- `go-stellar-sdk/xdr/xdr_generated.go:17747-17752` — `TransactionMetaV3` contains `TxChangesAfter` (where refunds lived pre-P23).
- `go-stellar-sdk/xdr/xdr_generated.go:18262-18270` — `TransactionMetaV4` contains `TxChangesAfter` but refund has moved out to `PostTxApplyFeeChanges`.
- `internal/transform/transaction_test.go:178-217` — Test case 4 expects `TxFeeMeta` to contain FeeChanges-only AND `ResourceFeeRefund=10705`. The test passes, confirming the design intent.

### Why It Failed

The hypothesis's expected behavior and mechanism are both incorrect. `tx_fee_meta` serializes `transaction.FeeChanges`, which is the **pre-transaction fee deduction** — it has never contained fee refund data in any protocol version. Before P23, refund data was in `UnsafeMeta.V3.TxChangesAfter` (exported as `tx_meta`). In P23+, it moved to `PostTxApplyFeeChanges` (a separate field on `LedgerTransaction`). The claim that refund data "moved out of `FeeChanges`" is false — refund data was never in `FeeChanges`. The existing test suite confirms that `tx_fee_meta` containing only the deduction is the intended behavior.

### Lesson Learned

`FeeChanges` and `PostTxApplyFeeChanges` represent different temporal phases of fee processing (deduction vs refund). `tx_fee_meta` maps to the deduction phase only and always has. When analyzing fee-related blob fields, verify the historical data lineage across protocol versions rather than assuming a field name implies comprehensive coverage. Note: there may be a separate valid observation that `PostTxApplyFeeChanges` is not exported as a raw blob for P23+ (it was previously recoverable from `tx_meta` via V3's `TxChangesAfter`), but that would be a different hypothesis with a different mechanism.
