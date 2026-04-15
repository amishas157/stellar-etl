# H002: `export-restored-keys` loses keys restored and removed in the same ledger

**Date**: 2026-04-15
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

A `restored_key` row should be exported whenever a ledger key is actually restored during a ledger, even if a later transaction in that same ledger removes the key again. The `restored_key` table is event-shaped output, so it should record the restoration event itself rather than only keys that survive to ledger close.

## Mechanism

`extractBatch()` compacts per-ledger changes before `export_ledger_entry_changes` inspects them for restored keys. With `SuppressRemoveAfterRestoreChange: false`, the upstream `ChangeCompactor` replaces a cached `LedgerEntryRestored` change with a `LedgerEntryRemoved` change when the same key is removed later in the ledger; the command then filters the compacted slice for `changeType == LedgerEntryRestored`, so the restoration disappears entirely from the exported `restored_key` stream.

## Trigger

1. Find or construct a ledger where a Soroban archived entry is restored by `RestoreFootprint`, then deleted again by a later transaction in the same ledger.
2. Run `stellar-etl export_ledger_entry_changes --export-restored-keys --start-ledger <L> --end-ledger <L>`.
3. The correct export should still contain one `restored_key` row for the restored ledger key.
4. The current path can emit no `restored_key` row at all because the compactor rewrites the event to `removed` before the command inspects it.

## Target Code

- `internal/input/changes.go:102-149` — compacts each ledger's changes before any restored-key filtering happens
- `internal/input/changes.go:104` — opts into `SuppressRemoveAfterRestoreChange: false`
- `cmd/export_ledger_entry_changes.go:111-126` — emits `restored_key` rows only when the compacted change type is `LedgerEntryRestored`
- `internal/transform/restored_key.go:11-47` — transformer only accepts `LedgerEntryRestored` inputs
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/change_compactor.go:249-259` — restored→removed transitions are rewritten to a plain removed change when suppression is disabled
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/change_compactor_test.go:491-508` — upstream test locks in that restored→removed yields only a removed change under the current config

## Evidence

The local export path does not inspect raw ledger changes; it only sees the compacted slice. The upstream compactor behavior is explicit and tested: under the exact config used here, a restored key that is later removed in the same ledger is no longer present as a `LedgerEntryRestored` record, and `TransformRestoredKey()` has no fallback path for removed changes.

## Anti-Evidence

If the project intended `restored_key` to mean "keys still present after all same-ledger compaction," then the current behavior would be consistent with that narrower interpretation. The flag name, table name, and transformer contract all read like event export rather than survivor snapshot, which makes that interpretation look weaker.
