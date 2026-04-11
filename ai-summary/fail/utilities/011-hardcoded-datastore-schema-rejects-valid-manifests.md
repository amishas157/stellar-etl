# H003: Hardcoded datastore schema rejects valid manifest schemas

**Date**: 2026-04-11
**Subsystem**: utilities
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a datastore already contains a manifest describing its ledger batching layout, `CreateLedgerBackend()` should trust that manifest and read the export successfully. A legitimate ledger-exporter dataset with non-default `ledgersPerBatch` / `batchesPerPartition` values should not become unreadable just because stellar-etl compiled in older defaults.

## Mechanism

`CreateDatastore()` hardcodes `LedgersPerFile=1` and `FilesPerPartition=64000` into `DataStoreConfig.Schema`. The SDK's `LoadSchema()` does not treat those values as hints: when a manifest exists, it converts the local config back into a manifest via `toDataStoreManifest()` and rejects any non-zero schema mismatch through `compareManifests()`. That means a valid datastore written with a different batching schema will fail before export starts, even though the manifest already contains the correct layout.

## Trigger

Point `--datastore-path` at a ledger-exporter bucket whose `.config.json` advertises `ledgersPerBatch != 1` or `batchesPerPartition != 64000`, then run any datastore-backed export such as `stellar-etl export_transactions -e 12345`.

## Target Code

- `internal/utils/main.go:CreateDatastore:990-1006` — hardcodes the local datastore schema to `1 / 64000`
- `internal/utils/main.go:CreateLedgerBackend:1021-1036` — forwards that fixed schema into `datastore.LoadSchema()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/support/datastore/configure.go:toDataStoreManifest:33-41` — includes local schema values in manifest comparison input
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/support/datastore/configure.go:LoadSchema:148-178` — rejects manifest/config schema mismatches instead of trusting the manifest

## Evidence

The utility layer always supplies non-zero schema values, so the SDK comparison path is always armed. `LoadSchema()` only bypasses schema comparison when the local values are zero; stellar-etl never allows that state because `CreateDatastore()` fills in both fields unconditionally.

## Anti-Evidence

This will not show up against datastores that happen to use the same `1 / 64000` layout as the current hardcoded defaults. It is therefore configuration-sensitive and may be invisible in the project's primary data lake until the exporter schema changes or an alternate dataset is used.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (related to but distinct from fail/001 which covered the overwritten error, not the hardcoded values)
**Failed At**: reviewer

### Trace Summary

Traced the full execution path from `CreateDatastore()` (hardcodes `LedgersPerFile=1, FilesPerPartition=64000`) through `CreateLedgerBackend()` into the SDK's `LoadSchema()`. Confirmed that `compareManifests()` on line 174 of `configure.go` would indeed reject a manifest with non-matching values. However, the error from `LoadSchema` on line 1034 of `main.go` is silently overwritten by line 1036's `:=` assignment — the same overwrite pattern documented in fail/001. The zero-value `DataStoreSchema{}` then fails in `NewBufferedStorageBackend` which rejects `LedgersPerFile <= 0`. The net result is a loud startup error, not silent data corruption.

### Code Paths Examined

- `internal/utils/main.go:CreateDatastore:994-1002` — hardcodes `LedgersPerFile: 1, FilesPerPartition: 64000` in `DataStoreConfig.Schema`
- `internal/utils/main.go:CreateLedgerBackend:1033-1036` — `LoadSchema` error on line 1034 is overwritten by `:=` on line 1036
- `configure.go:compareManifests:87-119` — compares fields only when `expected` value is non-zero; with hardcoded non-zero values, comparison is always armed
- `configure.go:LoadSchema:148-185` — when manifest exists and comparison fails, returns zero-value schema and error
- `configure.go:LoadSchema:174-178` — calls `compareManifests(toDataStoreManifest(cfg), manifest)` which would reject non-matching datastores

### Why It Failed

This describes an **operational limitation** (inability to read datastores with non-default schema), not a **data correctness bug**. The hypothesis correctly identifies that the hardcoded values would cause `compareManifests` to reject non-matching manifests. However:

1. **No silent data corruption**: The export fails with an error — it does not produce wrong output. The `NewBufferedStorageBackend` constructor rejects the zero-value schema that results from the overwritten `LoadSchema` error.
2. **Not silent**: A startup error is returned to the caller, preventing any data export.
3. **Not data loss**: No data is written incorrectly. The export simply doesn't run.

The objective framing requires findings that produce "incorrect but plausible output that downstream systems consume without error." This scenario produces NO output and returns an error, which is the opposite of silent corruption.

### Lesson Learned

A configuration rigidity that prevents reading valid data sources is a usability/operational concern, not a data correctness bug. For a finding to qualify under the data-integrity objective, it must produce wrong output — not merely fail to produce any output.
