# H011: Restored-Key Compaction Erases `LedgerEntryRestored` Markers

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Structural data corruption: restored-key rows would be silently omitted
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a ledger entry is restored and then updated again within the same ledger, `export_ledger_entry_changes --export-restored-keys` should still see a `LedgerEntryRestored` change and emit the corresponding `restored_key` row. Same-ledger follow-up updates should refresh the post-state without erasing the fact that the entry was restored in that ledger.

## Mechanism

At first glance, `extractBatch()` compacts every ledger through `ChangeCompactor` before the command checks `change.ChangeType == LedgerEntryRestored`. If compaction were to fold a restore+update sequence into a plain update, the `restored_key` exporter would silently skip a real restored key because `TransformRestoredKey()` rejects any non-restored change type.

## Trigger

Process a ledger containing a `LedgerEntryRestored` change followed by a `LedgerEntryUpdated` change for the same Soroban key while `--export-restored-keys` is enabled. The suspected failure mode was that compaction would leave only an `UPDATED` change and the restored-key row would disappear.

## Target Code

- `internal/input/changes.go:101-149` — compacts per-ledger changes before `export_ledger_entry_changes` inspects change types
- `internal/transform/restored_key.go:12-20` — `TransformRestoredKey()` requires `LedgerEntryChangeTypeLedgerEntryRestored`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/change_compactor.go:189-204` — update compaction logic for existing restored entries
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/change_compactor_test.go:474-489` — upstream test for restored-then-updated entries

## Evidence

The restored-key export path only emits rows for `LedgerEntryRestored`, so any compaction step that rewrote that state to `UPDATED` would definitely hide real restored keys. The `extractBatch()` compaction happens before the command's restored-key filter, which made this a plausible data-loss path on first read.

## Anti-Evidence

The upstream `ChangeCompactor` explicitly preserves the original change type when an existing restored entry receives an update: `addUpdatedChange()` keeps `existingChange.ChangeType` for restored entries instead of rewriting it to `UPDATED`. The corresponding upstream test asserts exactly that outcome, verifying that a restored entry followed by an update still emerges from compaction as `LedgerEntryRestored` with the newer post-state.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

`ChangeCompactor` is intentionally designed to retain the `LedgerEntryRestored` type across same-ledger updates. The compactor test suite covers this exact sequence, and the implementation keeps `existingChange.ChangeType` when the existing entry is restored, so the restored-key marker survives compaction.

### Lesson Learned

When a hypothesis depends on state-machine compaction rewriting change types, inspect the upstream compactor tests before escalating it. In this case the SDK already encodes the restored-then-updated contract explicitly, which is stronger evidence than the initial surface-level suspicion from the ETL call site.
