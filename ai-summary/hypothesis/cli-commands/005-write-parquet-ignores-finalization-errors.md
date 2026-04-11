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
