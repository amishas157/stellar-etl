# H003: Signer-deletion tombstones lose the removed signer's last weight

**Date**: 2026-04-15
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When an account update removes a signer, the synthetic `account_signers` tombstone row should preserve the removed signer's last configured weight, just like whole-account removal rows preserve pre-image signer weights. A signer deleted with weight `20` should therefore export a deleted row with `weight = 20`, `deleted = true`, and `ledger_entry_change = REMOVED`.

## Mechanism

`TransformSigners()` correctly preserves pre-image sponsorship and `last_modified_ledger` for signers that disappear during an account `UPDATED` change, but it hard-codes `Weight: 0` for those same synthetic deletion rows. That makes update-path tombstones inconsistent with the table's own whole-account removal path and destroys the last-known signer weight that downstream authorization and threshold audits need.

## Trigger

Export `account_signers` for a ledger where `SetOptions` removes a signer whose last configured weight is non-zero, such as a signer with weight `20`. The deleted row emitted from the update diff exports `weight = 0` instead of the removed signer's pre-state weight.

## Target Code

- `internal/transform/account_signer.go:34-52` — whole-account signer rows preserve each signer's actual weight from `SignerSummary()`
- `internal/transform/account_signer.go:57-85` — update-path synthetic deletion rows append pre-only signers with hard-coded `Weight: 0`
- `cmd/export_ledger_entry_changes.go:148-156` — exports `TransformSigners()` output directly into the live `account_signers` dataset

## Evidence

The update-specific deletion branch explicitly copies several pieces of pre-image state (`AccountID`, `Sponsor`, `LastModifiedLedger`) but replaces the removed signer's weight with a literal zero. The checked-in unit test for that path currently encodes the same behavior, expecting a signer deleted from pre-state weight `20` to emit a tombstone row with `Weight: 0`.

## Anti-Evidence

The current tests lock in `weight = 0` for update-path deletions, so this may reflect an intentional tombstone convention. But the same code already follows a last-known-state model for sponsor and `last_modified_ledger`, and the non-update removal path preserves real signer weights, which makes zeroing only the weight look like a local inconsistency rather than a coherent export contract.
