# H016: Missing datastore manifest does not silently misread non-default ledger batches

**Date**: 2026-04-11
**Subsystem**: utilities
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the datastore manifest is missing, backend construction should either infer the real batch layout correctly or fail before export starts. A ledger export should not silently read the wrong object files because `stellar-etl` fell back to hardcoded schema values that do not match the datastore's actual `ledgersPerBatch` / `batchesPerPartition`.

## Mechanism

`CreateDatastore()` hardcodes `LedgersPerFile=1` and `FilesPerPartition=64000`, and SDK `LoadSchema()` explicitly falls back to those local values when `.config.json` is missing. That initially looked like a silent-corruption path: a datastore with a different physical layout but no manifest might be read through the wrong schema and yield plausible ledger rows from the wrong objects.

## Trigger

Point the ETL at a datastore whose manifest is missing but whose actual layout uses non-default batching (for example, multiple ledgers per object), then run a normal datastore-backed export such as `export_transactions` or `export_ledgers`.

## Target Code

- `internal/utils/main.go:CreateDatastore:990-1006` — hardcodes fallback schema values
- `internal/utils/main.go:CreateLedgerBackend:1021-1036` — feeds those defaults into `datastore.LoadSchema()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/support/datastore/configure.go:155-167` — missing-manifest branch returns local schema unverified
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/support/datastore/schema.go:33-65` — object keys are derived directly from `LedgersPerFile` / `FilesPerPartition`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledgerbackend/ledger_buffer.go:171-177` — missing derived object keys surface as file-retrieval errors

## Evidence

The fallback branch in `LoadSchema()` is real, and `stellar-etl` always supplies non-zero schema values, so a missing manifest definitely reuses the hardcoded `1 / 64000` layout. Since datastore object names encode the partition and file start/end boundaries from that schema, a layout mismatch looks dangerous at first glance.

## Anti-Evidence

The same schema parameters are embedded into the object key itself, so a mismatched layout points at **different filenames**, not at a plausible neighboring ledger batch. The buffered backend then surfaces an `os.ErrNotExist`/download failure and aborts before exporting rows, making this a loud startup/read failure rather than silent wrong-data corruption.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

I could not construct a credible silent-corruption path from the missing-manifest fallback alone. With a real schema mismatch, `GetObjectKeyFromSequenceNumber()` derives different partition/file names, and the backend fails when those objects are absent instead of quietly reading a wrong-but-valid ledger batch.

### Lesson Learned

Schema fallback is only a data-integrity issue when a wrong schema can still resolve to existing objects that decode into plausible records. Here, the datastore naming scheme bakes the schema into the object path, so mismatches tend to fail loudly at object lookup time.
