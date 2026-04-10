# H004: JSON exporters report success after `Close()` writeback failures

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: Data loss or silent partial uploads
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If finalizing the JSON output file fails, the export command should surface that error and stop before printing success stats or uploading the path. A completed export should mean the JSONL file was fully committed to storage.

## Mechanism

The one-shot export commands call `outFile.Close()` and ignore its returned error, then immediately continue to `MaybeUpload(...)` or exit successfully. On filesystems that surface delayed allocation or writeback failures at close time, the command can treat a truncated or partially committed file as successful output and optionally upload it to cloud storage anyway.

## Trigger

Run `export_ledgers`, `export_transactions`, `export_contract_events`, or `export_token_transfer` on a filesystem that returns an error from `Close()` after the row writes have already returned successfully, especially with cloud upload enabled.

## Target Code

- `cmd/export_ledgers.go:63-68` — ignores `outFile.Close()` result and proceeds to upload
- `cmd/export_transactions.go:56-61` — same pattern
- `cmd/export_contract_events.go:57-65` — same pattern
- `cmd/export_token_transfers.go:57-62` — same pattern

## Evidence

Each command calls `outFile.Close()` as a bare statement rather than `if err := outFile.Close(); err != nil { ... }`. The export flow then prints stats and invokes `MaybeUpload(...)`, so the close result is the only missing gate between a failed final flush and a success-shaped completion path.

## Anti-Evidence

On healthy local disks, `Close()` usually succeeds, so the bug hides in routine runs. It needs a close-time filesystem error to manifest, but when it does, the current command paths have no way to distinguish a complete file from a failed final commit.
