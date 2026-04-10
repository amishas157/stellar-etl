# H005: `WriteParquet()` drops footer and close errors after the last row write

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: Data loss or silent partial uploads
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Parquet exports should fail loudly if writing the footer or closing the file fails. Callers should only proceed to upload or keep the output path when `WriteParquet()` has confirmed the file is fully finalized and readable.

## Mechanism

`WriteParquet()` only checks errors from `writer.Write(record.ToParquet())`. The function defers both `writer.WriteStop()` and `parquetFile.Close()` without inspecting their returned errors, even though `WriteStop()` is the step that writes the Parquet footer. If the failure happens during final footer flush or close, the command proceeds as though the Parquet file succeeded and can upload a truncated or unreadable object.

## Trigger

Run any parquet-capable export on a filesystem that errors during footer write or final close, such as a near-full disk or failing network mount, after the per-record writes have already succeeded.

## Target Code

- `cmd/command_utils.go:162-180` — `WriteParquet()` defers `writer.WriteStop()` and `parquetFile.Close()` without checking their errors
- `cmd/export_contract_events.go:63-65` — representative caller that uploads the Parquet path immediately after `WriteParquet()` returns

## Evidence

`go doc github.com/xitongsys/parquet-go/writer.ParquetWriter.WriteStop` documents that `WriteStop()` "Write[s] the footer and stop[s] writing" and returns an `error`. The repo code drops that error on the floor with `defer writer.WriteStop()`, so the finalization step that makes the file a valid Parquet object is never validated.

## Anti-Evidence

If no footer or close error occurs, the file is fine and the bug stays invisible. This only manifests when failure is delayed until finalization, but that is precisely the case where the current implementation has no protection.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `WriteParquet()` in `cmd/command_utils.go:162-180`. Confirmed that `defer writer.WriteStop()` and `defer parquetFile.Close()` both discard their returned errors. Verified the `WriteStop()` implementation in `parquet-go@v1.6.2/writer/writer.go` — it calls `Flush(true)` to flush remaining buffered rows, then writes column indices, offset indices, and the Parquet footer to the file, returning errors at each step. The `Close()` method on the local file writer delegates to `os.File.Close()`, also returning error. Both errors are silently dropped by the deferred calls, creating an inconsistency with the function's own error-handling pattern which uses `cmdLogger.Fatal()` for all other failure modes.

### Code Paths Examined

- `cmd/command_utils.go:162-180` — `WriteParquet()` function. Lines 167 and 173 use bare `defer` calls that drop errors from `parquetFile.Close()` and `writer.WriteStop()` respectively. Lines 164-166, 170-172, and 176-178 use `cmdLogger.Fatal()` for file creation, writer creation, and record write errors.
- `parquet-go@v1.6.2/writer/writer.go:WriteStop()` — Confirmed signature `func (pw *ParquetWriter) WriteStop() error`. Implementation calls `pw.Flush(true)`, writes ColumnIndexes, OffsetIndexes, and footer metadata to `pw.PFile`, each step returning error on failure.
- `parquet-go-source@v0.0.0-20240122235623-d6294584ab18/local/local.go:Close()` — Confirmed signature `func (self *LocalFile) Close() error`, delegates to `self.File.Close()`.
- `cmd/export_contract_events.go:63-65` — Representative caller: `WriteParquet()` on line 64, immediately followed by `MaybeUpload()` on line 65. If `WriteStop()` silently failed, the incomplete file is uploaded.
- `cmd/command_utils.go:123-146` — `MaybeUpload()` reads the file at the given path and uploads to GCS, with no validation that the file is a valid Parquet object.

### Findings

**Error-handling inconsistency confirmed.** `WriteParquet()` uses `cmdLogger.Fatal()` (which calls `os.Exit(1)`) for three error conditions: file creation, writer creation, and per-record writes. But the two most critical finalization steps — footer writing and file close — silently discard errors. This creates a gap where the function can return normally despite producing an invalid file.

**`WriteStop()` performs multiple critical I/O operations:**
1. `Flush(true)` — flushes any remaining buffered row data to the file
2. Column index writes — serializes and writes column index entries for each column chunk
3. Offset index writes — serializes and writes offset index entries
4. Footer metadata write — writes the Parquet footer that makes the file readable

If any of these steps fail, the file is truncated or missing its footer. Without the footer, the Parquet file is unreadable by any standard Parquet reader (including BigQuery).

**Callers unconditionally upload after `WriteParquet()` returns.** All 7 export commands that support `--write-parquet` call `MaybeUpload()` immediately after `WriteParquet()` returns. If the footer write failed silently, a truncated/unreadable Parquet file is uploaded to GCS.

**Distinct from related reviewed findings:**
- Reviewed H004 (json-exporters-ignore-close-failures): covers JSON `outFile.Close()` errors, not Parquet
- Reviewed H004 (parquet-upload-before-write): covers upload ordering, not error swallowing within `WriteParquet()`
- Reviewed H005 (export-entry-swallows-write-errors): covers `ExportEntry()` JSON `Write()` errors, not Parquet

### PoC Guidance

- **Test file**: `cmd/command_utils_test.go` (create or append)
- **Setup**: Create a `WriteParquet()` test that writes records to a Parquet file, then verifies the file is valid. To demonstrate the error-dropping bug, use a mock `source.ParquetFile` implementation whose `Write()` or `Close()` methods return an error during the footer-writing phase. Alternatively, use `parquet-go`'s own reader to verify the output file has a valid footer.
- **Steps**:
  1. Call `WriteParquet()` with valid data and a valid path — confirm it produces a readable Parquet file (baseline).
  2. Instrument or mock the underlying file writer to fail during `WriteStop()` (e.g., close the underlying fd before `WriteStop()` runs, or use a size-limited temp filesystem). Observe that `WriteParquet()` returns without error/fatal.
  3. Attempt to read the resulting Parquet file with `parquet-go`'s reader — it should fail with a missing/corrupt footer error.
- **Assertion**: Demonstrate that `WriteParquet()` can return normally (no Fatal) while producing an invalid Parquet file. After the fix, `WriteStop()` and `Close()` errors should trigger `cmdLogger.Fatal()` consistent with the function's existing error-handling pattern.
