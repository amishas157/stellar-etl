# H005: `WriteParquet` drops footer and flush failures from `WriteStop`

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If parquet finalization fails while flushing row groups, writing column indexes, writing the footer, or closing the destination file, the export command should surface that failure and stop before claiming success or uploading the file. A truncated parquet file should never be treated as a completed export.

## Mechanism

`WriteParquet` defers both `writer.WriteStop()` and `parquetFile.Close()` but ignores their returned errors. In parquet-go, `WriteStop()` performs the final `Flush(true)` plus footer and index writes, so late `ENOSPC`, quota, or filesystem errors can happen after all per-row `writer.Write(...)` calls already succeeded. Those failures are silently discarded, and the caller continues as though the parquet file were valid.

## Trigger

Run any parquet-writing export on a filesystem that allows row writes but fails during footer finalization, such as a nearly full disk or quota-limited mount. The command will return from `WriteParquet` without error even though the parquet footer write failed and the output file is incomplete.

## Target Code

- `cmd/command_utils.go:WriteParquet:162-180` — deferred `WriteStop()` and `Close()` return values are ignored
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/writer/writer.go:129-202` — `WriteStop()` performs final flush and footer/index writes and returns I/O errors
- `cmd/export_assets.go:79-81` — representative caller assumes `WriteParquet` succeeded and proceeds to upload
- `cmd/export_contract_events.go:63-65` — another caller treats normal return as a completed parquet export

## Evidence

The only checked write path inside `WriteParquet` is the per-record `writer.Write(...)` loop. The deferred finalization steps are the ones that actually emit the parquet footer and trailer magic, and parquet-go explicitly returns errors from those writes, but the helper never captures or propagates them.

## Anti-Evidence

Mid-stream row-write failures are still handled correctly because the loop calls `cmdLogger.Fatal` immediately on `writer.Write(...)` errors. The missed corruption window is specifically the late finalization phase after all rows have already been accepted.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`WriteParquet` (cmd/command_utils.go:162-180) uses `defer writer.WriteStop()` and `defer parquetFile.Close()`, discarding both error returns. In Go, `defer f()` silently drops return values. The parquet-go `WriteStop()` method (writer.go:129-202) performs six distinct I/O operations—Flush, ColumnIndex writes, OffsetIndex writes, footer serialization, footer size, and trailing "PAR1" magic—each returning errors that propagate to the caller. Since `WriteParquet` has no return value and defers finalization without error capture, any I/O failure during these critical steps is invisible to the five callers that proceed to `MaybeUpload` immediately after.

### Code Paths Examined

- `cmd/command_utils.go:WriteParquet:162-180` — confirmed: `defer writer.WriteStop()` on line 173 discards the error; `defer parquetFile.Close()` on line 167 discards the error; function signature returns nothing
- `writer/writer.go:WriteStop:129-202` (parquet-go v1.6.2) — confirmed: performs `Flush(true)`, writes ColumnIndex buffers, writes OffsetIndex buffers, writes footer, writes footer size, writes "PAR1" trailer; each step returns error on I/O failure
- `local/local.go:Write` (parquet-go-source) — confirmed: delegates to `os.File.Write`, propagates errors
- `cmd/export_assets.go:80-81` — confirmed: calls `WriteParquet` then `MaybeUpload` with no error check between them
- `cmd/export_effects.go:67-68` — confirmed: same pattern
- `cmd/export_contract_events.go:64-65` — confirmed: same pattern
- `cmd/export_ledger_entry_changes.go:371-372` — confirmed: same pattern

### Findings

The bug is confirmed. There is a clear inconsistency in error handling within `WriteParquet`:
1. **Row writes**: `writer.Write()` errors trigger `cmdLogger.Fatal` — process terminates immediately.
2. **Finalization**: `writer.WriteStop()` errors are silently discarded via `defer`.
3. **File close**: `parquetFile.Close()` errors are silently discarded via `defer`.

Without a valid footer (written by `WriteStop`), a parquet file is structurally invalid and unreadable by any standard parquet reader (Spark, BigQuery, pyarrow). Five callers proceed to `MaybeUpload` after `WriteParquet` returns, potentially uploading a corrupt file to GCS.

This is distinct from existing findings:
- Success 001 covers upload-before-write ordering (different bug)
- Success 002 covers ExportEntry swallowing JSON write errors (different function, different I/O path)

### PoC Guidance

- **Test file**: `cmd/command_utils_test.go` or a new `cmd/write_parquet_poc_test.go`
- **Setup**: Create a mock `source.ParquetFile` implementation that accepts all `Write` calls during row writing but returns an error during the footer-phase writes (i.e., after N successful Write calls, start returning `os.ErrClosed` or a custom error). Alternatively, use a real file on a tmpfs with a small size limit.
- **Steps**: Call `WriteParquet` with valid data and the error-injecting file. Observe that the function returns without panic/fatal.
- **Assertion**: After `WriteParquet` returns, attempt to read the output file with a parquet reader (`parquet-go/reader`). The read should fail because the footer is missing or truncated. This proves the function silently produced an invalid file. Alternatively, a static analysis test can verify that `defer writer.WriteStop()` is used without error capture (similar to the source-scanning approach in the existing PoC for success/001).
