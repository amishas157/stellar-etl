# H003: `export_assets --captive-core` aborts on the first asset row because history input carries an empty `LedgerCloseMeta`

**Date**: 2026-04-11
**Subsystem**: utilities
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_assets --captive-core` should export each discovered asset with the real `closed_at` timestamp and `ledger_sequence` of the ledger that first introduced it. Even when the command reads from a history-style helper, it should either supply equivalent ledger metadata to `TransformAsset()` or fail cleanly with a returned error before starting the transform loop.

## Mechanism

`GetPaymentOperationsHistoryArchive()` explicitly sets `LedgerCloseMeta: xdr.LedgerCloseMeta{}` for every candidate asset operation. `TransformAsset()` then calls `utils.GetCloseTime(lcm)` and `utils.GetLedgerSequence(lcm)`, and both utility helpers dereference `lcm.LedgerHeaderHistoryEntry()`. In the zero-value union, `V == 0` but `V0` is unset, so the XDR helper panics with `arm V0 is not set`; the command never receives an ordinary error and the export aborts mid-run instead of emitting asset rows.

## Trigger

Run `stellar-etl export_assets --captive-core -s <start> -e <end>` over any legitimate ledger range containing at least one payment or manage-sell-offer operation that introduces an asset row. The first call to `TransformAsset()` on that range should hit the empty-`LCM` panic path.

## Target Code

- `cmd/export_assets.go:30-33` - selects `GetPaymentOperationsHistoryArchive(...)` when `UseCaptiveCore` is true
- `internal/input/assets_history_archive.go:33-39` - appends `AssetTransformInput` rows with `LedgerCloseMeta: xdr.LedgerCloseMeta{}`
- `internal/transform/asset.go:45-50` - unconditionally reads close time and ledger sequence from the provided `LCM`
- `internal/utils/main.go:GetCloseTime/GetLedgerSequence:968-976` - forwards to `lcm.LedgerHeaderHistoryEntry()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/ledger_close_meta.go:8-18` - zero-value `LedgerCloseMeta` routes to `MustV0()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:20121-20128` - `MustV0()` panics when arm `V0` is unset

## Evidence

The history-archive asset helper leaves a code comment acknowledging that it does not support `LCM`, but the downstream transform still treats `LCM` as mandatory. There is no guard, alternate source of close time / ledger sequence, or panic recovery in `export_assets`, so the empty-union access is a straight crash path once any asset candidate is transformed.

## Anti-Evidence

If the selected ledger range contains no qualifying payment or manage-sell-offer operations, the transform loop never touches the bad `LCM` and the command may appear healthy. The normal non-`--captive-core` path also avoids the issue by passing real `LedgerCloseMeta` values from `CreateLedgerBackend()`.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of ai-summary/success/utilities/003-captive-core-flag-routes-to-history-archive.md
**Failed At**: reviewer

### Trace Summary

The hypothesis describes the `export_assets --captive-core` panic path where `GetPaymentOperationsHistoryArchive()` sets `LedgerCloseMeta: xdr.LedgerCloseMeta{}` and `TransformAsset()` subsequently panics when calling `utils.GetCloseTime(lcm)` which dereferences the unset V0 arm. This exact mechanism was already investigated, confirmed, and published as part of a broader finding covering both `export_assets` and `export_ledgers` under the `--captive-core` flag.

### Code Paths Examined

- `cmd/export_assets.go:30-33` — confirms `UseCaptiveCore` routes to `GetPaymentOperationsHistoryArchive()`
- `internal/input/assets_history_archive.go:33-39` — confirms `LedgerCloseMeta: xdr.LedgerCloseMeta{}` is explicitly set
- `internal/transform/asset.go:45-50` — confirms unconditional reads of close time and ledger sequence from LCM
- `internal/utils/main.go:968-976` — confirms `GetCloseTime()`/`GetLedgerSequence()` forward to `lcm.LedgerHeaderHistoryEntry()`

### Why It Failed

This is an exact duplicate of the already-confirmed and published finding in `ai-summary/success/utilities/003-captive-core-flag-routes-to-history-archive.md`. That finding explicitly documents: "`export_assets --captive-core` feeds `xdr.LedgerCloseMeta{}` into `TransformAsset`, which panics when `utils.GetCloseTime()` calls `LedgerHeaderHistoryEntry()` on the unset union arm." The success file covers the identical code paths (`cmd/export_assets.go` → `GetPaymentOperationsHistoryArchive()` → `LedgerCloseMeta{}` → `TransformAsset` panic), the identical trigger (`--captive-core` flag), and includes a complete PoC test (`TestZeroLCMCrashesTransformAsset`).

### Lesson Learned

The success file 003 in `utilities/` covers both the `export_ledgers` silent zeroing and the `export_assets` panic as two manifestations of the same root cause (inverted `--captive-core` flag routing). New hypotheses about individual symptoms of this root cause should be checked against existing multi-symptom findings.
