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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

I traced the full data flow from the XDR `LedgerCloseMetaV2` through the SDK's `LedgerTransactionReader.Read()` into `TransformLedgerTransaction()`. The SDK extracts `PostTxApplyFeeProcessing` from `lcmV2.TxProcessing[i].PostTxApplyFeeProcessing` into `LedgerTransaction.PostTxApplyFeeChanges` (ledger_transaction_reader.go:90-101). `TransformLedgerTransaction` serializes `FeeChanges` and `UnsafeMeta` but never touches `PostTxApplyFeeChanges`. For P23+ (TransactionMetaV4), the `TxChangesAfter` field inside `UnsafeMeta` is empty — fee refund balance changes live exclusively in `PostTxApplyFeeChanges`. Since no exported field carries this data, it is silently dropped from the raw export.

### Code Paths Examined

- `internal/transform/ledger_transaction.go:32-34` — `tx_fee_meta` is base64(FeeChanges) only; PostTxApplyFeeChanges is never referenced
- `internal/transform/ledger_transaction.go:27-30` — `tx_meta` is base64(UnsafeMeta); for v4 meta, TxChangesAfter is empty per P23 design
- `internal/transform/schema.go:86-94` — `LedgerTransactionOutput` struct has no field for PostTxApplyFeeChanges
- `internal/transform/transaction.go:195-202` — higher-level `TransformTransaction()` already correctly appends PostTxApplyFeeChanges to TxChangesAfter for P23+ refund calculation
- SDK `ingest/ledger_transaction_reader.go:90-101` — `PostTxApplyFeeChanges` populated from `lcmV2.TxProcessing[i].PostTxApplyFeeProcessing`
- SDK `ingest/ledger_transaction.go:639-642` — upstream `GetFeeChangesIncludingRefund()` explicitly appends PostTxApplyFeeChanges
- SDK `xdr/xdr_generated.go:19002-19005` — `TransactionResultMetaV1` defines `PostTxApplyFeeProcessing` as a distinct field in the V2 ledger close meta

### Findings

The finding is confirmed: `TransformLedgerTransaction()` exports only two XDR components related to fees — `FeeChanges` (pre-apply fee deductions) into `tx_fee_meta` and `UnsafeMeta` (transaction metadata) into `tx_meta`. For pre-P23 transactions, the fee refund balance changes lived inside `UnsafeMeta`'s `TxChangesAfter`, so the export was complete. Starting with P23 (TransactionMetaV4), `TxChangesAfter` is empty and refund changes moved to `PostTxApplyFeeChanges` on the `LedgerTransaction` struct — a separate field that `TransformLedgerTransaction` never reads.

The higher-level `TransformTransaction()` already handles this correctly (lines 195-202), providing strong evidence this is a known semantic change that the raw export hasn't been updated for.

Severity is **High** (structural data corruption — missing data element) rather than Critical because: (1) the data is absent, not incorrect; (2) the higher-level transaction export handles this correctly; (3) the raw export's purpose is to provide base64 XDR blobs, and this is a missing blob rather than a corrupted one.

### PoC Guidance

- **Test file**: `internal/transform/ledger_transaction_test.go`
- **Setup**: Create a mock `ingest.LedgerTransaction` with `UnsafeMeta` set to a V4 `TransactionMeta` (empty `TxChangesAfter`) and a non-empty `PostTxApplyFeeChanges` containing a balance change entry. Also create a valid `xdr.LedgerHeaderHistoryEntry` with `LedgerVersion >= 23`.
- **Steps**: Call `TransformLedgerTransaction(transaction, lhe)` and decode the returned `TxFeeMeta` from base64 back to `xdr.LedgerEntryChanges`.
- **Assertion**: Assert that the decoded `TxFeeMeta` does NOT contain the PostTxApplyFeeChanges entries (demonstrating the data loss). Then assert that `PostTxApplyFeeChanges` on the input struct is non-empty, proving the data existed but was dropped.
