# H005: `WriteParquet()` treats footer/close failures as success and can upload truncated parquet

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If Parquet finalization fails while flushing row groups, writing indexes/footer, or closing the file handle, the export command should surface that error and avoid reporting success or uploading the malformed artifact.

## Mechanism

`WriteParquet()` defers both `parquetFile.Close()` and `writer.WriteStop()` without inspecting their return values. In `parquet-go`, `WriteStop()` is the step that flushes pending pages and writes the footer/index blocks; a late filesystem error at that stage is silently discarded, after which callers continue as though the Parquet file is complete and may upload a truncated or unreadable file.

## Trigger

Run any `--write-parquet` export on storage that starts failing only during final flush, footer, or close — for example, a nearly full disk that accepts row writes but errors once `WriteStop()` serializes the footer and trailing metadata.

## Target Code

- `cmd/command_utils.go:162-180` — defers `parquetFile.Close()` and `writer.WriteStop()` without checking either error
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/writer/writer.go:128-202` — `WriteStop()` returns errors from `Flush()` and footer/index `PFile.Write()` calls
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/example/local_flat.go:51-60` — upstream examples explicitly check the `WriteStop()` error before declaring success

## Evidence

The production helper only handles errors from `writer.NewParquetWriter()` and per-record `writer.Write()`. The library itself documents the real finalization contract in code: `WriteStop()` returns `error`, and upstream examples treat that return as mandatory to check.

## Anti-Evidence

Immediate write-path failures are still surfaced because `writer.Write()` is checked inside the loop. This hypothesis only concerns late failures that happen after the last row was accepted but before the file becomes a valid finished Parquet artifact.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`WriteParquet()` in `cmd/command_utils.go:162-180` uses `defer writer.WriteStop()` and `defer parquetFile.Close()`, discarding both error returns. The parquet-go library's `WriteStop()` method (`writer/writer.go:128-202`) performs substantial finalization work — flushing remaining pages via `Flush(true)`, serializing column indexes, offset indexes, the Thrift-encoded footer, footer size, and the trailing `PAR1` magic bytes — returning errors from every `PFile.Write()` call. Because the defer discards these errors, `WriteParquet()` returns as though the file is complete. At least four export commands (`export_assets`, `export_effects`, `export_contract_events`, `export_ledger_entry_changes`) then call `MaybeUpload()` immediately after, uploading a potentially truncated parquet file to GCS.

### Code Paths Examined

- `cmd/command_utils.go:162-180` — `WriteParquet()` defers `WriteStop()` and `Close()` without capturing error returns; function signature is `void` so cannot propagate errors
- `parquet-go/writer/writer.go:128-202` — `WriteStop()` calls `Flush(true)` then writes column indexes, offset indexes, footer, footer size, and magic bytes, returning error at each stage
- `parquet-go/writer/writer.go:246-300` — `Flush()` calls `flushObjs()` then writes row group chunk data with error checking
- `cmd/export_effects.go:67-68` — calls `WriteParquet()` then `MaybeUpload()` in sequence
- `cmd/export_assets.go:80-81` — same Write-then-Upload pattern
- `cmd/export_contract_events.go:64-65` — same pattern
- `cmd/export_ledger_entry_changes.go:371-372` — same pattern

### Findings

The bug is real: Go's `defer` statement discards the return value of the deferred function call. Since `WriteParquet()` has no error return, there is no mechanism for callers to learn that finalization failed. The `WriteStop()` method in parquet-go performs the most critical part of parquet file construction — without the footer and trailing magic bytes, the file is unreadable by any parquet consumer. If the underlying filesystem returns an error during any of the ~6 write operations in `WriteStop()`, the error is silently dropped and the export continues to upload the corrupt artifact.

This is distinct from the previously confirmed finding `003-stale-parquet-upload-before-regeneration` (which concerns upload ordering) and `008-export-entry-swallows-write-errors` (which concerns JSON output error handling in `ExportEntry()`).

### PoC Guidance

- **Test file**: `cmd/command_utils_test.go` (or create one if it doesn't exist)
- **Setup**: Create a mock `ParquetFile` implementation that returns `nil` from `Write()` during normal row writes but returns an error when `WriteStop()` calls its final `Write()` (e.g., track call count and fail on the Nth write). Alternatively, use a wrapper around `local.NewLocalFileWriter` that returns write errors after a byte threshold.
- **Steps**: Call `WriteParquet()` with a valid schema and a small dataset using the error-injecting file writer. Verify that `WriteParquet()` returns without panicking or fataling (confirming the error is swallowed).
- **Assertion**: After `WriteParquet()` returns, read the output file and verify it is NOT a valid parquet file (missing footer/magic bytes). This proves a caller would proceed to upload corrupt data.
