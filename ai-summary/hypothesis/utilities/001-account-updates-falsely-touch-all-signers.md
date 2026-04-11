# H001: Unrelated account updates falsely mark every signer as updated

**Date**: 2026-04-10
**Subsystem**: utilities / data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --export-accounts` processes an `AccountEntry` update whose signer set did not change, the `account_signers` output should either emit no signer rows at all or emit rows only for signers whose own fields changed. A `SetOptions` operation that only updates unrelated account metadata such as thresholds or home domain should not fabricate signer updates.

## Mechanism

`TransformSigners()` appends one output row for every signer in the post-update account before it compares `Pre` vs `Post`. The later diff logic only synthesizes removed rows; it never suppresses unchanged post-state signers. As a result, any account update that leaves the signer set untouched still exports one `ledger_entry_change = UPDATED` row per signer, making downstream consumers believe every signer changed in that ledger.

## Trigger

Process an `AccountEntry` `LedgerEntryUpdated` change where `Pre` and `Post` contain the same signer list and sponsor-per-signer map, but some unrelated account field changes (for example `HomeDomain`, thresholds, or flags). The correct `account_signers` delta is empty, but the current code emits rows for all current signers.

## Target Code

- `internal/transform/account_signer.go:14-52` — unconditionally appends one row per post-state signer with the parent account change type
- `internal/transform/account_signer.go:54-87` — only diffs for deleted signers; unchanged signers are never filtered out

## Evidence

The main loop at lines 35-52 iterates over `accountEntry.SignerSummary()` for every updated account and writes `LedgerEntryChange: uint32(changeType)` into every row. The only special handling in the update branch is the second loop at lines 67-86, which adds synthetic removal rows for missing signers but never skips unchanged signers.

## Anti-Evidence

If `account_signers` were intentionally modeled as a full post-update snapshot table, repeating unchanged signers on every account update would be acceptable. But the table already carries `ledger_entry_change` and `deleted`, and the recent deletion hotfix explicitly adds synthetic row-level removals, which strongly suggests downstream consumers expect signer-level deltas rather than unconditional snapshots.
