# H001: export_assets Captive-Core Path Zeroes Asset Close Metadata

**Date**: 2026-04-10
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`stellar-etl export_assets` should emit each asset row with the real ledger close time and ledger sequence from the ledger that contained the triggering payment or manage-sell-offer operation, regardless of whether the export uses datastore or `--use-captive-core`.

## Mechanism

When `commonArgs.UseCaptiveCore` is true, `cmd/export_assets.go` routes the export through `GetPaymentOperationsHistoryArchive()` instead of the normal `GetPaymentOperations()` path. That history-archive reader hard-codes `LedgerCloseMeta` to `xdr.LedgerCloseMeta{}`, but `TransformAsset()` derives `closed_at` and `ledger_sequence` exclusively from that `LedgerCloseMeta`, so the export can silently emit zero-valued metadata (`1970-01-01T00:00:00Z` / ledger sequence `0`) for otherwise valid asset rows.

## Trigger

Run `stellar-etl export_assets --use-captive-core --start-ledger <L> --end-ledger <R>` over any range containing at least one payment or manage-sell-offer operation that introduces an asset. Compare the exported `closed_at` and `ledger_sequence` fields against the same range without `--use-captive-core` or against the source ledger headers.

## Target Code

- `cmd/export_assets.go:30-45` — selects the history-archive reader when `UseCaptiveCore` is true, then feeds its output into `TransformAsset`
- `internal/input/assets_history_archive.go:13-45` — populates `AssetTransformInput` with an empty `xdr.LedgerCloseMeta{}`
- `internal/transform/asset.go:45-50` — derives `ClosedAt` and `LedgerSequence` only from `LedgerCloseMeta`
- `internal/utils/main.go:968-975` — converts `LedgerCloseMeta` into close time and ledger sequence without any alternate source

## Evidence

The normal reader in `internal/input/assets.go` preserves the real `ledger` value in `LedgerCloseMeta`, but the history-archive variant explicitly cannot and stores a zero-value `LedgerCloseMeta` instead. The export path still calls the same transformer, so the metadata fields are sourced from the empty struct rather than from `LedgerSeqNum` or the history-archive ledger header already available in the reader.

## Anti-Evidence

`asset_code`, `asset_issuer`, `asset_type`, and `asset_id` are still extracted directly from the operation body, so only the ledger metadata columns appear corrupted. If downstream consumers ignore `closed_at` and `ledger_sequence`, the bad rows can look superficially valid.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

The hypothesis correctly identifies that `GetPaymentOperationsHistoryArchive()` (assets_history_archive.go:38) hard-codes `LedgerCloseMeta: xdr.LedgerCloseMeta{}` with all-nil pointer fields. However, the predicted behavior — silently emitting zero-valued metadata — is wrong. The actual behavior is a nil-pointer panic that crashes the process before any data is exported.

### Code Paths Examined

- `internal/input/assets_history_archive.go:33-39` — confirms `LedgerCloseMeta: xdr.LedgerCloseMeta{}` is constructed with V=0, V0=nil
- `internal/transform/asset.go:45-46` — calls `utils.GetCloseTime(lcm)` which invokes `lcm.LedgerHeaderHistoryEntry()`
- `go-stellar-sdk/xdr/ledger_close_meta.go:8-12` — `LedgerHeaderHistoryEntry()` switches on `V`; for V=0 calls `l.MustV0()`
- `go-stellar-sdk/xdr/xdr_generated.go:MustV0/GetV0` — `GetV0()` executes `result = *u.V0`; since V0 is nil, this is a nil-pointer dereference → **panic**

### Why It Failed

The zero-value `xdr.LedgerCloseMeta{}` does NOT produce zero-valued metadata fields. Instead, it causes a nil-pointer panic in `GetV0()` when trying to dereference the nil `V0` pointer (`*u.V0`). The process crashes immediately upon encountering the first payment/sell-offer operation, preventing ANY data from being exported. Since no output is produced, there is no silent data corruption — the bug manifests as an obvious crash, not as plausible-but-wrong data. This falls outside the objective's scope of "silent data corruption: bugs that produce incorrect but plausible output."

### Lesson Learned

When analyzing XDR union types with zero-value constructors, check whether the union's pointer arms are nil. A zero-value `LedgerCloseMeta{}` sets `V=0` but leaves `V0=nil`, which panics on access rather than returning zero-valued inner fields. Always trace through the SDK's accessor chain (MustVN → GetVN → dereference) to determine whether the failure mode is a crash or silent corruption.
