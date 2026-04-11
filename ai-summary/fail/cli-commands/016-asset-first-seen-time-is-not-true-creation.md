# H016: `export_assets` timestamps may look like asset creation times but are only first-seen-in-range metadata

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: Low
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `history_assets.closed_at` and `history_assets.ledger_sequence` are intended to mean the true on-chain creation time of an asset, then rerunning `export_assets` on a later range should not shift those values forward just because the exporter first re-encountered the asset later.

## Mechanism

`TransformAsset()` stamps each row with the close time and ledger sequence of the operation currently being scanned, and `export_assets` keeps only the first matching row per `AssetID` within the requested range. At first glance, that makes later-range exports look like they are re-creating the asset at a newer ledger rather than preserving an authoritative creation timestamp.

## Trigger

Run `export_assets` on a ledger range that starts after an asset has already existed on-chain but still contains a later operation referencing that asset. The exported row will carry the later range's `closed_at` and `ledger_sequence`.

## Target Code

- `cmd/export_assets.go:39-69` — command deduplicates by `AssetID` only within the current export range
- `internal/transform/asset.go:45-50` — row timestamps come from the currently scanned ledger close meta
- `README.md:236-245` — command description is brief enough that the row timestamp semantics are easy to over-read

## Evidence

The transform has no access to any prior-global asset history; it only sees the operation currently being exported. That means the row's temporal fields necessarily describe the observation chosen by this export run, not some independent asset-issuance event.

## Anti-Evidence

There is no dedicated on-chain "asset created" event or ledger-entry source wired into this command. The implementation is consistently an operation-derived asset-reference export, so there is no clearly correct alternative creation timestamp available from the current pipeline inputs.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The code never claims to export authoritative asset-creation chronology; it exports asset references discovered from scanned operations and stamps them with the ledger that exposed that reference during the current run. Without a separate state/history source for "true creation," the later-range timestamp is not provably wrong under the command's current contract.

### Lesson Learned

When a dataset is derived from observed operations rather than ledger-entry lifecycle state, "first seen in this export" metadata can look suspicious without actually violating the command's semantics. A viable finding here needs a stronger contract breach than "these timestamps are not global creation times."
