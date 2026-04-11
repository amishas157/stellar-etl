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
