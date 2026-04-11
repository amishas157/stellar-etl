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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated. Fail entry 054+055 was about redefining `tx_fee_meta` semantics; this hypothesis correctly targets the absence of a dedicated column, which is the exact reframing 054+055's lesson recommended.

### Trace Summary

Traced the complete data flow from XDR protocol definition through SDK reader to the transform function. `TransactionResultMetaV1` (used in `LedgerCloseMetaV2.TxProcessing`) has `PostTxApplyFeeProcessing` as a field structurally separate from `TxApplyProcessing` (which maps to `UnsafeMeta`). The SDK's `LedgerTransactionReader.Read()` populates `LedgerTransaction.PostTxApplyFeeChanges` from `lcmV2.TxProcessing[i].PostTxApplyFeeProcessing`. `TransformLedgerTransaction()` serializes `UnsafeMeta` → `tx_meta` and `FeeChanges` → `tx_fee_meta` but never reads `PostTxApplyFeeChanges`. Pre-P23, fee refund changes lived in `UnsafeMeta.V3.TxChangesAfter` (part of `tx_meta`); P23+ moved them to `PostTxApplyFeeChanges` (not part of `UnsafeMeta`), creating a silent data regression.

### Code Paths Examined

- `internal/transform/ledger_transaction.go:13-58` — `TransformLedgerTransaction()` serializes `transaction.UnsafeMeta` (line 27) and `transaction.FeeChanges` (line 32) but has no reference to `PostTxApplyFeeChanges`
- `internal/transform/schema.go:86-94` — `LedgerTransactionOutput` struct has fields for `TxMeta`, `TxFeeMeta`, `TxEnvelope`, `TxResult`, `TxLedgerHistory`, `ClosedAt` — no field for post-apply fee changes
- `internal/transform/transaction.go:195-202` — Sibling `TransformTransaction()` explicitly handles P23+ by appending `transaction.PostTxApplyFeeChanges` to `metav4.TxChangesAfter` for refund computation, proving codebase awareness of this field
- `go-stellar-sdk/xdr/xdr_generated.go:18988-19007` — `TransactionResultMetaV1` defines `PostTxApplyFeeProcessing LedgerEntryChanges` as structurally separate from `TxApplyProcessing TransactionMeta`
- `go-stellar-sdk/xdr/xdr_generated.go:18245-18272` — `TransactionMetaV4` contains `TxChangesAfter` but NOT `PostTxApplyFeeProcessing`; the latter is at the `TransactionResultMetaV1` level, not within `TransactionMeta`
- `go-stellar-sdk/ingest/ledger_transaction_reader.go:95-101` — Reader extracts `PostTxApplyFeeChanges` from `lcmV2.TxProcessing[i].PostTxApplyFeeProcessing`, a separate field from `TxApplyProcessing` (which becomes `UnsafeMeta`)

### Findings

1. **Data regression confirmed**: Pre-P23, fee refund balance changes were embedded in `TransactionMeta.V3.TxChangesAfter`, which is part of `UnsafeMeta` and serialized into `tx_meta`. P23+ moved these changes to `PostTxApplyFeeProcessing`, a field at the `TransactionResultMetaV1` level — structurally outside `TransactionMeta`. The `tx_meta` blob no longer contains refund changes for P23+ ledgers.

2. **No alternate export path exists**: `LedgerTransactionOutput` has no field for post-apply fee changes. The schema was not updated for P23. The data is available in the SDK's `LedgerTransaction.PostTxApplyFeeChanges` field but is silently dropped.

3. **Sibling was updated, this was not**: `TransformTransaction()` (lines 195-200) explicitly handles `PostTxApplyFeeChanges` by appending it to `TxChangesAfter` for resource-fee-refund computation. `TransformLedgerTransaction()` was not similarly updated, confirming this is an oversight rather than a design choice.

4. **Impact**: Downstream consumers of the `ledger_transaction` raw export cannot reconstruct fee refund amounts from the exported blobs for P23+ Soroban transactions with non-zero refunds. The exported data looks valid but is silently incomplete.

### PoC Guidance

- **Test file**: `internal/transform/ledger_transaction_test.go`
- **Setup**: Construct an `ingest.LedgerTransaction` with `UnsafeMeta` containing `TransactionMetaV4` (V=4) and populate `PostTxApplyFeeChanges` with a non-empty `LedgerEntryChanges` slice (e.g., a fee account balance state change representing a refund). Call `TransformLedgerTransaction()`.
- **Steps**: (1) Create the test transaction with V4 meta and non-empty `PostTxApplyFeeChanges`. (2) Call `TransformLedgerTransaction()`. (3) Decode the returned `TxMeta` (base64 → `TransactionMeta`). (4) Decode the returned `TxFeeMeta` (base64 → `LedgerEntryChanges`). (5) Check all output fields for the presence of the post-apply fee changes.
- **Assertion**: Assert that the post-apply fee changes from `PostTxApplyFeeChanges` do NOT appear in any of `TxMeta`, `TxFeeMeta`, or any other output field — confirming the data is silently dropped. The test demonstrates that the `LedgerTransactionOutput` cannot preserve fee refund XDR for P23+ transactions.
