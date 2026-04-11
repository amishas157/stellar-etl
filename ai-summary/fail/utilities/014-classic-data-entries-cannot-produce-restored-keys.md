# H014: `export-restored-keys` does not currently miss reachable classic `DATA` restores

**Date**: 2026-04-10
**Subsystem**: utilities
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If classic `LedgerEntryTypeData` entries could be archived and later restored, `export_ledger_entry_changes --export-restored-keys` should emit `restored_key` rows for those restores instead of silently dropping them.

## Mechanism

`extractBatch()` hardcodes a tracked type list and explicitly skips `LedgerEntryTypeData`. Because `TransformRestoredKey()` is otherwise type-agnostic, this initially looked like a generic restored-key coverage gap that would make the command miss valid restored `DATA` keys.

## Trigger

Process a ledger containing a `LedgerEntryRestored` change for a classic `LedgerEntryTypeData` entry while `--export-restored-keys` is enabled.

## Target Code

- `internal/input/changes.go:88-97` — tracked change types exclude `LedgerEntryTypeData`
- `internal/input/changes.go:125-135` — explicitly skips `LedgerEntryTypeData`
- `internal/transform/restored_key.go:12-47` — restored-key transform itself does not depend on the ledger entry subtype

## Evidence

The input layer comment explicitly calls out `LedgerEntryTypeData` as intentionally untracked, so any generic restored-key export must rely on the tracked-type list to make those rows visible. On first inspection that looks like a silent omission for one ledger entry family.

## Anti-Evidence

Classic account `DATA` entries are not subject to Soroban TTL, archival, or `RestoreFootprint` restoration semantics. The only archived/restored entry families are Soroban contract-storage-related entries, so a `LedgerEntryRestored` change for classic `LedgerEntryTypeData` is not a realistic live trigger.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The tracked-type gap exists in code, but the missing classic `DATA` restore is not a reachable on-chain event under current Stellar archival semantics. Without a concrete restorable `DATA` entry type, this does not create live data loss today.

### Lesson Learned

Before treating a hardcoded type filter as a restored-key omission, verify that the omitted entry family can actually participate in archival/restore. Soroban restore semantics do not apply to every ledger entry type that happens to be representable in XDR.
