# H001: Ignored datastore schema error does not silently corrupt exports

**Date**: 2026-04-10
**Subsystem**: utilities
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the ETL cannot load the datastore schema, backend creation should fail loudly instead of continuing with a zero-value schema that points at the wrong ledger-exporter files. The caller should receive an error before any JSON or Parquet export begins.

## Mechanism

`CreateLedgerBackend` assigns `schema, err = datastore.LoadSchema(...)` and then immediately overwrites `err` with the result of `ledgerbackend.NewBufferedStorageBackend(...)` without checking the schema-load error first. That looked like it might allow a zero-value `DataStoreSchema` to slip through and silently mis-map ledger file boundaries.

## Trigger

Use datastore mode with a missing or unreadable manifest so `datastore.LoadSchema` returns an error before export startup.

## Target Code

- `internal/utils/main.go:CreateLedgerBackend:1021-1037` — overwrites the `LoadSchema` error before checking it
- `github.com/stellar/go-stellar-sdk/ingest/ledgerbackend/buffered_storage_backend.go:NewBufferedStorageBackend:47-58` — validates the schema before constructing the backend

## Evidence

The utility code really does discard the first `err` value, so the call site looks suspicious on inspection. `DataStoreSchema` zero values also matter, because object-key derivation depends on `LedgersPerFile` and `FilesPerPartition`.

## Anti-Evidence

`NewBufferedStorageBackend` rejects any schema with `LedgersPerFile <= 0`, so the zero-value schema produced by a failed `LoadSchema` call cannot become a live backend. `LoadSchema` also has an explicit fallback path that returns a populated schema when the manifest is missing but the local config is complete.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The downstream backend constructor turns the discarded schema-load failure into a hard startup error by rejecting zero-valued schema settings, so this path does not silently emit wrong ledger data.

### Lesson Learned

An overwritten intermediate error is only a real data-integrity bug if the next layer accepts the partially initialized value. Here, the SDK validates the schema early enough that exports stop instead of misreading files.
