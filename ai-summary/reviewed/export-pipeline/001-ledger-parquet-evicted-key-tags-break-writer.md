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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-13
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the parquet-go schema initialization path for `LedgerOutputParquet`. The two `[]string` fields (`EvictedLedgerKeysType`, `EvictedLedgerKeysHash`) use scalar tags `type=BYTE_ARRAY, convertedtype=UTF8` without `valuetype` or `repetitiontype=REPEATED`. When parquet-go's `NewSchemaHandlerFromStruct` encounters a `[]string` with default RepetitionType (REQUIRED=0, not REPEATED=2), it enters the LIST handler, which calls `GetValueTagMap()` to extract the element type from the `valuetype` tag key. Since `valuetype` is never set, the element's `Type` is an empty string, and `NewSchemaElementFromTagMap` fails at `parquet.TypeFromString("")` with the error `"type : not a valid Type string"`. This error propagates through `WriteParquet` → `cmdLogger.Fatal`, killing the process before any rows are written.

### Code Paths Examined

- `internal/transform/schema_parquet.go:27-28` — `EvictedLedgerKeysType` and `EvictedLedgerKeysHash` are `[]string` with scalar `type=BYTE_ARRAY` tags, missing `valuetype`/`repetitiontype=REPEATED`
- `internal/transform/schema_parquet.go:59` — `ExtraSigners` is the correct pattern: `[]string` with `type=MAP, convertedtype=LIST, valuetype=BYTE_ARRAY, valueconvertedtype=UTF8`
- `internal/transform/schema_parquet.go:66,362-363` — `SorobanResourcesArchivedEntries`, `BucketListSizeWindow`, `LiveSorobanStateSizeWindow` use the other correct pattern: `repetitiontype=REPEATED`
- `parquet-go@v1.6.2/schema/schemahandler.go:297-334` — Slice with non-REPEATED RepetitionType enters LIST path, calls `GetValueTagMap()` for element type
- `parquet-go@v1.6.2/common/common.go:553-567` — `GetValueTagMap()` copies `src.ValueType` to `res.Type`; with no `valuetype` tag, this is `""`
- `parquet-go@v1.6.2/common/common.go:293-308` — `NewSchemaElementFromTagMap()` calls `parquet.TypeFromString(info.Type)` which fails on empty string
- `cmd/command_utils.go:169-171` — `WriteParquet()` calls `cmdLogger.Fatal` on writer creation error
- `cmd/export_ledgers.go:70-72` — Ledger export unconditionally uses `LedgerOutputParquet` for `--write-parquet`

### Findings

The bug is confirmed. `LedgerOutputParquet` has two `[]string` fields with parquet tags that use the scalar `BYTE_ARRAY` pattern instead of either:
1. The LIST pattern: `type=MAP, convertedtype=LIST, valuetype=BYTE_ARRAY, valueconvertedtype=UTF8` (used by `ExtraSigners`)
2. The REPEATED pattern: `type=BYTE_ARRAY, repetitiontype=REPEATED` (used by numeric slice fields)

The codebase itself has both correct patterns for slice fields, confirming these two tags are outliers. The `WriteParquet` function fatals on the schema error, so `export_ledgers --write-parquet` is completely broken — zero rows are ever emitted.

### PoC Guidance

- **Test file**: `internal/transform/schema_parquet_test.go` (create if needed; or add to an existing parquet test file)
- **Setup**: Import `parquet-go/writer` and `parquet-go/source/local`. Create a temp file.
- **Steps**: Call `writer.NewParquetWriter(tempFile, new(transform.LedgerOutputParquet), 1)`
- **Assertion**: Assert that the returned error is non-nil and contains `"not a valid Type string"`. This directly demonstrates that the schema cannot initialize. Optionally, also show that after fixing the tags to use `type=MAP, convertedtype=LIST, valuetype=BYTE_ARRAY, valueconvertedtype=UTF8`, the writer initializes successfully.
