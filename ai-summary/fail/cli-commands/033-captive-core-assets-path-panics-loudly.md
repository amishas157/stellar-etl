# H033: `export_assets --captive-core` emits wrong `closed_at` and `ledger_sequence`

**Date**: 2026-04-14
**Subsystem**: cli-commands
**Severity**: High
**Impact**: wrong asset timestamps and ledger sequence
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_assets` is run with `--captive-core`, each exported asset row should still contain the real ledger close time and ledger sequence for the operation that introduced the asset.

## Mechanism

The command's `--captive-core` branch routes through `GetPaymentOperationsHistoryArchive()`, which hard-codes `LedgerCloseMeta{}` into every `AssetTransformInput`. That initially looks like a silent-zero-value path for `closed_at` and `ledger_sequence`, because `TransformAsset()` derives both fields from `utils.GetCloseTime(lcm)` and `utils.GetLedgerSequence(lcm)`.

## Trigger

Run `export_assets --captive-core` over any ledger range containing at least one qualifying `Payment` or `ManageSellOffer` operation.

## Target Code

- `cmd/export_assets.go:30-34` — `--captive-core` selects `GetPaymentOperationsHistoryArchive()`
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:30-39` — injects zero-value `LedgerCloseMeta`
- `internal/transform/asset.go:45-50` — derives `ClosedAt` and `LedgerSequence` from that `LedgerCloseMeta`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/ledger_close_meta.go:8-18` — zero-value `LedgerCloseMeta.LedgerHeaderHistoryEntry()` dereferences the selected arm

## Evidence

The history-archive helper really does pass an empty `LedgerCloseMeta`, and `TransformAsset()` has no alternate source for close time or ledger sequence. On paper that suggests a silent corruption path any time the command uses `--captive-core`.

## Anti-Evidence

The zero-value `LedgerCloseMeta` does not quietly return zero timestamps; its helper methods dereference nil union arms and panic. So the path fails loudly on the first qualifying asset instead of producing plausible-but-wrong rows.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-14
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This routing bug is real, but its manifestation is a crash, not silent corruption. The empty `LedgerCloseMeta` panics before any asset row can be serialized with wrong values.

### Lesson Learned

When a command passes a zero-value XDR union into a transform, verify whether the transform returns default values or dereferences a nil arm. Nil-arm panics push the issue out of the "silent data corruption" class even if the routing bug itself is real.
