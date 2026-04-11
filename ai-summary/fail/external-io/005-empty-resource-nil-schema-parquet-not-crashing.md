# H005: Empty-resource parquet writes crash when `export_ledger_entry_changes` leaves the schema nil

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a selected ledger-entry-change resource has zero rows in a batch, the exporter should either emit a valid empty parquet artifact or skip that parquet file cleanly; a nil schema should not itself crash the batch.

## Mechanism

I investigated whether `exportTransformedData()` would fatal on empty resource buckets because `parquetSchema` is discovered lazily inside the per-row switch. If true, any batch with zero rows for a selected resource would hit `WriteParquet(..., nil)` and abort the export even though empty batches are a normal condition.

## Trigger

Run `export_ledger_entry_changes --export-config-settings true --write-parquet` on a batch that has no config-setting rows, such as the existing `empty_config` test range.

## Target Code

- `cmd/export_ledger_entry_changes.go:304-372` — initializes `parquetSchema` to nil and still calls `WriteParquet()` after the row loop
- `cmd/export_ledger_entry_changes_test.go:139-143` — existing test coverage exercises a zero-row config-setting batch

## Evidence

The control flow really does leave `parquetSchema` unset when a selected resource produces no rows in that batch. On paper, that looks like it should be unsafe for the Parquet writer.

## Anti-Evidence

I manually checked `parquet-go` with `NewParquetWriter(file, nil, 1)`, `WriteStop()`, and `NewParquetReader(file, nil, 1)`, and all three calls succeeded for a zero-row file. That means the library tolerates this odd empty-file case instead of crashing.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The nil-schema path is only dangerous when real rows need to be serialized. For genuinely empty resource buckets, `parquet-go` produces and reads back an empty file without error, so the broad “every empty batch crashes” claim does not hold.

### Lesson Learned

For this exporter, separate empty-resource behavior from unsupported non-empty resource types. The empty case is tolerated by the library; the higher-value bugs are the paths where JSON rows exist but the Parquet switch never appends them.
