# H005: GCS uploads leak source file descriptors and can drop later batch uploads

**Date**: 2026-04-13
**Subsystem**: external-io
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each call to `MaybeUpload()` / `GCS.UploadTo()` should close the local source file after `io.Copy()` finishes so the number of live descriptors stays bounded across long-running or multi-batch exports.

## Mechanism

`GCS.UploadTo()` opens the local file with `os.Open(path)` and never closes the returned reader. Because the function also deletes the path afterward, the leak is easy to miss on Unix: the pathname disappears, but the file descriptor stays live until GC closes it. In continuous or batched exports with cloud upload enabled, those leaked descriptors can accumulate until later uploads fail to open their local files or other export files, causing missing downstream artifacts.

## Trigger

Run a long-lived export with `--cloud-provider gcp`, especially `export_ledger_entry_changes` over many batches or continuous mode (`end-ledger=0`), so the process performs many uploads in a single lifetime.

## Target Code

- `cmd/upload_to_gcs.go:32-35` — opens the source file with `os.Open(path)` and keeps the handle alive
- `cmd/upload_to_gcs.go:52-73` — copies, verifies, deletes the local path, and returns without `reader.Close()`
- `cmd/command_utils.go:123-146` — dispatches uploads whenever cloud output is enabled
- `cmd/export_ledger_entry_changes.go:368-372` — a high-frequency caller that can upload many files in one process

## Evidence

There is no `defer reader.Close()` or any other close path in `UploadTo()`. The prior external-io records already established that missing closes on export files can exhaust file descriptors under sustained operation; this upload helper introduces the same class of leak on the cloud-upload side for every command that calls `MaybeUpload()`.

## Anti-Evidence

Short one-shot commands may exit before the leak becomes user-visible, and Go's GC may eventually reclaim some descriptors. The bug needs a process that performs many uploads in one lifetime, so it is strongest on continuous or highly batched export paths rather than a single small export.
