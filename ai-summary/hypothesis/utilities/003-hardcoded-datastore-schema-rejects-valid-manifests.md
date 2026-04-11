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
