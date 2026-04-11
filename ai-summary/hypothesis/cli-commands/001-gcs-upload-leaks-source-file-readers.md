# H001: GCS upload leaks source file descriptors across uploaded exports

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each export that uploads a local file to GCS should close the local file reader after `io.Copy` completes so long-running exports can keep opening, uploading, and deleting later output files. A successful upload should not leave an open descriptor behind for the uploaded file.

## Mechanism

`GCS.UploadTo` opens the local file with `os.Open(path)` and streams it into the GCS writer, but never closes that `reader`. Commands that call `MaybeUpload` repeatedly — especially `export_ledger_entry_changes`, which uploads one JSON file and optionally one parquet file per resource per batch — accumulate leaked descriptors until later exports or uploads fail to open their next file, silently dropping later output ranges after earlier uploads appeared successful.

## Trigger

Run any command with `--cloud-provider gcp`, especially `export_ledger_entry_changes` over many batches or a command that uploads both JSON and parquet files in the same process. After enough uploaded files, the process approaches the OS open-file limit because each uploaded file remains open even after `deleteLocalFiles(path)` removes the pathname.

## Target Code

- `cmd/upload_to_gcs.go:25-73` — `UploadTo` opens the local file reader, copies it, and never closes it
- `cmd/command_utils.go:123-146` — `MaybeUpload` invokes `UploadTo` for every uploaded output file
- `cmd/export_ledger_entry_changes.go:368-372` — long-running batch exporter repeatedly uploads per-resource JSON/parquet files
- `cmd/export_assets.go:77-81` — one-shot exporters also hit the same leaked-reader path for both JSON and parquet uploads

## Evidence

The upload path has `reader, err := os.Open(path)` at line 32 and no matching `defer reader.Close()` anywhere in the function. The same function then deletes the uploaded pathname via `deleteLocalFiles(path)` at line 71, so the leak persists only as an unlinked-but-open descriptor that remains invisible to later cleanup.

## Anti-Evidence

Short runs that upload only one or two files may exit before the descriptor leak becomes observable. The corruption mechanism depends on repeated uploads in a single process rather than on any specific ledger content.
