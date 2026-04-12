# H005: JSON exporters ignore `Close()` failures and can upload truncated files as success

**Date**: 2026-04-12
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: silent JSON artifact truncation
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the final `Close()` on an export's JSON output file fails, the command should surface a non-zero error and refuse to report success or upload that artifact. The correct output value is the fully flushed JSONL file; a late flush failure should never be treated as equivalent to a complete export.

## Mechanism

Every JSON export command calls `outFile.Close()` and discards the returned error. That leaves a blind spot after the last `Write`/`WriteString`: late filesystem failures from quota-limited, network-backed, or delayed-write storage can turn a supposedly complete JSONL file into a truncated artifact, yet the command still logs success-style byte counts and often proceeds into `MaybeUpload()`.

This is a distinct integrity gap from the already-published per-write error swallowing in `ExportEntry()`. Even if every row write reports success, the final flush can still fail, and the current command layer has no path to prevent downstream consumers from receiving an incomplete JSON file.

## Trigger

Run any JSON exporter (for example `export_effects`, `export_assets`, or `export_contract_events`) on storage that accepts the individual writes but fails on final flush during `close(2)`, such as a quota-limited mount, some FUSE/NFS backends, or a filesystem that reports delayed allocation failures only at close. The command will ignore the `Close()` error, print normal stats, and can immediately upload the incomplete file.

## Target Code

- `cmd/export_assets.go:58-81` — ignores `outFile.Close()` and then logs/upload proceeds
- `cmd/export_effects.go:44-68` — same pattern on a multi-row JSON exporter
- `cmd/export_contract_events.go:42-65` — same pattern before optional upload/parquet
- `cmd/export_ledgers.go:50-72` — representative single-row-per-input exporter with the same unchecked close
- `cmd/export_transactions.go:43-65` — same unchecked close pattern
- `cmd/export_operations.go:43-65` — same unchecked close pattern
- `cmd/export_trades.go:47-70` — same unchecked close pattern
- `cmd/export_token_transfers.go:46-62` — same unchecked close pattern
- `cmd/export_ledger_transaction.go:42-56` — same unchecked close pattern
- `cmd/upload_to_gcs.go:32-69` — opens and uploads the file immediately after the unchecked close paths report success

## Evidence

Each command calls `outFile.Close()` as a bare statement and then immediately logs success or starts upload work; none branches on the returned error. The already-published `get_ledger_range_from_times` finding proved this repository accepts silent file-output corruption as an integrity issue when write/close errors are ignored, and the main export commands currently have the same unchecked close sink on their JSON artifacts.

## Anti-Evidence

`os.File.Close()` failures are rarer than ordinary `Write` failures on local disks, so the trigger usually needs quota pressure or a filesystem that reports delayed-write errors late. But the command layer is explicitly where the final success/failure decision is made, and today it has no way to detect or stop a late-flush corruption path once row writes appear to have succeeded.
