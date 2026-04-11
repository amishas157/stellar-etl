# H003: Synthetic signer deletions keep the stale pre-update `last_modified_ledger`

**Date**: 2026-04-10
**Subsystem**: utilities / data-transform
**Severity**: Medium
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `TransformSigners()` synthesizes a deleted signer row for a signer removed during an account update, the exported row should identify the ledger that actually removed the signer as the row's latest modification. A signer removal that occurs in ledger `L` should not look as if its last modification happened in some older ledger merely because the parent account entry was last touched earlier.

## Mechanism

The update-only deletion branch explicitly copies `ledgerChange.Pre.LastModifiedLedgerSeq` into `preLastModifiedLedger` and uses that stale value on every synthetic removal row. The row simultaneously carries the current `ledger_sequence` and `closed_at`, so the export produces an internally inconsistent deletion record: the row says the signer was removed now, but its `last_modified_ledger` points at the previous account version instead of the removal ledger.

## Trigger

Process an `AccountEntry` `LedgerEntryUpdated` change where a signer present in `Pre` is absent in `Post`, and the account's `LastModifiedLedgerSeq` changes between the two versions. The exported synthetic removal row should reflect the current ledger as the latest signer modification, but the current code keeps the stale pre-update value.

## Target Code

- `internal/transform/account_signer.go:65-66` — stores `preLastModifiedLedger` from the pre-update account entry
- `internal/transform/account_signer.go:75-85` — writes that stale value into synthetic removal rows

## Evidence

The checked-in deletion test intentionally sets different `LastModifiedLedgerSeq` values on `Pre` and `Post`, then asserts that the deleted signer row keeps the pre-state value while surviving signers use the post-state value. That demonstrates the stale-ledger behavior is deliberate in code today, not just theoretical.

## Anti-Evidence

Some existing tables reuse `last_modified_ledger` from the removed pre-state ledger entry rather than the ledger that emitted the removal. But these signer deletions are synthetic child-row records created inside an updated parent account entry, so there is no standalone signer ledger entry whose historical last-modified value needs to be preserved; the only actual signer-row event being exported is the deletion happening now.
