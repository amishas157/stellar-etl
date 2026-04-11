# H001: Unchanged signer rows are emitted as `UPDATED` on targeted signer edits

**Date**: 2026-04-11
**Subsystem**: utilities
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a ledger change modifies only one signer on an existing account, the `account_signers` export should emit rows only for the signer entries whose lifecycle actually changed. Unchanged sibling signers and the unchanged master key should not receive fresh `UPDATED` rows, because there is no prior signer row to revise for them in that ledger.

## Mechanism

`AccountSignersChanged()` is a coarse account-level gate: once any signer membership, weight, or sponsorship change is detected, `export_ledger_entry_changes` calls `TransformSigners()` for the whole account. `TransformSigners()` then appends **every** post-state signer with `ledger_entry_change = UPDATED` and only adds synthetic `REMOVED` rows for pre-only signers, so a one-signer edit fans out into multiple plausible-but-wrong `UPDATED` rows for unaffected signers.

## Trigger

Run `export_ledger_entry_changes --export-accounts` over a ledger containing a `SetOptions` update that changes exactly one signer on a multi-signer account, such as:

1. Change signer A's weight while signer B and the master key remain unchanged.
2. Change signer A's sponsorship while signer B and the master key remain unchanged.
3. Delete signer B from an account that still retains signer A and the master key.

In each case, inspect the emitted `account_signers` rows for the unaffected surviving signers.

## Target Code

- `internal/utils/main.go:AccountSignersChanged:1051-1119` — reduces signer activity to a single account-level boolean
- `cmd/export_ledger_entry_changes.go:148-156` — exports signer rows whenever that coarse gate is true
- `internal/transform/account_signer.go:34-52` — emits every post-state signer with the parent account change type
- `internal/transform/account_signer.go:57-86` — only synthesizes deletion rows for pre-only signers; unchanged survivors are never filtered out
- `internal/transform/account_signer_test.go:256-295` — current regression test explicitly expects unchanged surviving signers to be emitted as `UPDATED`

## Evidence

The production exporter already treats `account_signers` as a per-signer lifecycle stream: it emits synthetic `REMOVED` rows for deleted signers and stores per-row `ledger_entry_change` / `deleted` fields. Despite that, the first loop in `TransformSigners()` blindly appends the full post-state signer set once `AccountSignersChanged()` says that **some** signer changed, and the checked-in test fixture for a single-signer deletion bakes in `UPDATED` rows for the unchanged master key and unchanged surviving signer.

## Anti-Evidence

If downstream consumers intentionally treat `account_signers` as a full post-update snapshot for any signer-changing account, the extra rows may look unsurprising. But that interpretation conflicts with the existing synthetic `REMOVED` logic and with the row-level lifecycle fields already present in the exported schema.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — substantially addressed by success/utilities/002-added-signers-are-mislabeled-as-updated
**Failed At**: reviewer

### Trace Summary

Traced the full path from `AccountSignersChanged()` (lines 1051-1119 in `internal/utils/main.go`) through `export_ledger_entry_changes.go` (lines 148-156) into `TransformSigners()` (lines 14-96 in `internal/transform/account_signer.go`). Confirmed that the first loop (lines 35-52) emits ALL post-state signers with the parent account's `changeType`, including signers whose weight and sponsorship are identical in pre and post states. The second loop (lines 57-86) synthesizes REMOVED rows for pre-only signers. The test at lines 256-295 explicitly expects unchanged surviving signers to appear as UPDATED.

### Code Paths Examined

- `internal/utils/main.go:AccountSignersChanged:1051-1119` — correctly detects any signer membership, weight, or sponsorship change at the account level
- `internal/utils/main.go:ExtractEntryFromChange:854-865` — returns Post entry and `LedgerEntryChangeTypeLedgerEntryUpdated` for update changes
- `internal/transform/account_signer.go:TransformSigners:14-96` — emits all post-state signers with parent change type, then synthesizes REMOVED rows for pre-only signers
- `internal/transform/account_signer_test.go:256-295` — test fixture explicitly expects unchanged master key (weight=2) and unchanged signer A (weight=10) to be emitted as UPDATED
- `cmd/export_ledger_entry_changes.go:148-156` — calls TransformSigners when AccountSignersChanged is true, appends all returned rows

### Why It Failed

1. **Describes working-as-designed behavior.** The regression test at `account_signer_test.go:256-295` explicitly expects unchanged surviving signers to receive UPDATED rows. The test comments ("Master key remains: LedgerEntryChange=UPDATED" and "Signer A remains: LedgerEntryChange=UPDATED") confirm this is intentional, not accidental.

2. **Already addressed by prior finding.** Success finding `002-added-signers-are-mislabeled-as-updated` analyzed the same `TransformSigners` code path and explicitly recommended "retain UPDATED for signers present in both states" in its suggested fix. That investigation already considered whether unchanged survivors should be filtered or relabeled, and concluded the snapshot-on-change model is acceptable for signers present in both pre and post states.

3. **The ETL uses a snapshot-on-change model.** When any signer changes on an account, the exporter emits the complete post-state signer set. This is a valid design choice for ETL tools — downstream consumers always see the full signer set after any change, simplifying state reconstruction. The REMOVED rows exist because deleted signers have no post-state representation and must be synthesized; this doesn't imply that surviving signers should be filtered.

### Lesson Learned

When a prior successful finding explicitly addresses a related concern in its suggested fix (e.g., "retain UPDATED for signers present in both states"), that constitutes a design decision that has been reviewed and accepted. Hypotheses that re-litigate the same design choice from a different angle should be checked against prior findings' fix recommendations, not just their titles.
