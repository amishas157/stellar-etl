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
