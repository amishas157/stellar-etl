# H001: Ledger Parquet export cannot initialize once evicted-key columns are in schema

**Date**: 2026-04-13
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: empty results from broken control flow
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_ledgers --write-parquet` should be able to initialize a Parquet writer and
emit the same ledger rows that the JSON path already exports, including the
`evicted_ledger_keys_type` and `evicted_ledger_keys_hash` arrays for ledgers
whose close meta contains evicted Soroban keys.

## Mechanism

`LedgerOutputParquet` models both evicted-key columns as `[]string`, but their
tags use a scalar `BYTE_ARRAY` shape instead of parquet-go's list grammar. As a
result, `WriteParquet()` fails at `writer.NewParquetWriter(...)` before any row
is written, so the Parquet path for `export_ledgers` aborts instead of producing
ledger rows.

## Trigger

1. Run `stellar-etl export_ledgers --write-parquet --start-ledger <s> --end-ledger <e>`.
2. Reach the Parquet branch in `cmd/export_ledgers.go`.
3. The writer initialization for `transform.LedgerOutputParquet` fails with
   `failed to create schema from tag map: type : not a valid Type string`.

## Target Code

- `internal/transform/schema_parquet.go:23-28` — `LedgerOutputParquet` declares
  `EvictedLedgerKeysType` and `EvictedLedgerKeysHash` as `[]string` with scalar
  `BYTE_ARRAY` tags
- `internal/transform/parquet_converter.go:27-56` — `LedgerOutput.ToParquet()`
  passes both string slices straight into the Parquet struct
- `cmd/command_utils.go:162-172` — `WriteParquet()` initializes the writer and
  fatals immediately if schema creation fails
- `cmd/export_ledgers.go:70-72` — the ledger command always uses this schema for
  `--write-parquet`

## Evidence

A direct in-repo `writer.NewParquetWriter(..., new(transform.LedgerOutputParquet), 1)`
reproduction fails immediately with `type : not a valid Type string`. Reducing
the schema to either `EvictedLedgerKeysType` alone or `EvictedLedgerKeysHash`
alone reproduces the same failure, which points at those two tags rather than at
the rest of the ledger schema.

## Anti-Evidence

The failure is Parquet-only: JSON ledger export still works, and the rest of the
ledger Parquet fields are ordinary scalars. If parquet-go had accepted these
slice tags as an idiosyncratic shorthand, this would be a false alarm — but the
writer-init failure makes the breakage concrete on the live export path.
