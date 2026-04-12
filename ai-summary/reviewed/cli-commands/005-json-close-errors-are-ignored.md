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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

All 9 main export commands (`export_assets`, `export_effects`, `export_ledgers`, `export_transactions`, `export_operations`, `export_trades`, `export_contract_events`, `export_token_transfers`, `export_ledger_transaction`) call `outFile.Close()` as a bare statement, discarding the returned error. After the unchecked Close(), each command logs success statistics (byte counts, transform stats) and proceeds into `MaybeUpload()`. This is distinct from success/002 (per-row write errors in `ExportEntry`) and success/007 (which only covers `get_ledger_range_from_times`). The `export_ledger_entry_changes` command's `exportTransformedData` function has a separate known issue where it never calls Close() at all (success/004).

### Code Paths Examined

- `cmd/export_assets.go:72` — `outFile.Close()` bare call, error discarded; line 73 logs byte count, line 77 calls `MaybeUpload`
- `cmd/export_effects.go:59` — `outFile.Close()` bare call, error discarded; line 60 logs byte count, line 64 calls `MaybeUpload`
- `cmd/export_ledgers.go:63` — `outFile.Close()` bare call, error discarded; line 64 logs byte count, line 68 calls `MaybeUpload`
- `cmd/export_transactions.go:56` — `outFile.Close()` bare call, error discarded; line 57 logs byte count, line 61 calls `MaybeUpload`
- `cmd/export_operations.go:56` — `outFile.Close()` bare call, error discarded; line 57 logs byte count, line 61 calls `MaybeUpload`
- `cmd/export_trades.go:61` — `outFile.Close()` bare call, error discarded; line 62 logs byte count, line 66 calls `MaybeUpload`
- `cmd/export_contract_events.go:57` — `outFile.Close()` bare call, error discarded; line 59 calls `PrintTransformStats`, line 61 calls `MaybeUpload`
- `cmd/export_token_transfers.go:57` — `outFile.Close()` bare call, error discarded; line 58 logs byte count, line 62 calls `MaybeUpload`
- `cmd/export_ledger_transaction.go:51` — `outFile.Close()` bare call, error discarded; line 52 logs byte count, line 56 calls `MaybeUpload`
- `cmd/command_utils.go:55-86` — `ExportEntry` itself (related but separate finding, success/002)

### Findings

The pattern is universal and consistent across all 9 main export commands: `outFile.Close()` is called as a bare statement with no error capture. In Go, `(*os.File).Close()` returns an error that can indicate a flush failure on the underlying file descriptor. When the kernel reports a delayed-write error only at `close(2)` time (common with NFS, FUSE mounts, quota-limited filesystems), the JSONL file may be truncated or incomplete, yet the command:

1. Logs success-style byte counts
2. Calls `PrintTransformStats` (reporting successful transforms)
3. Calls `MaybeUpload` which can upload the truncated file to GCS

This creates a silent data integrity gap: downstream consumers (BigQuery loaders, analytics pipelines) receive and ingest a truncated JSONL file with no indication of failure.

This is the same class of bug as success/007 (`get_ledger_range_from_times` ignoring Close errors) but affects a different and larger set of commands — the 9 primary data export commands that produce the actual blockchain data artifacts.

### PoC Guidance

- **Test file**: `cmd/data_integrity_poc_test.go` (existing test file used by success/002 and success/007)
- **Setup**: Build the `stellar-etl` binary. Create a temp directory for output. Use `ulimit -f 0` or a pipe with a closed read end to deterministically trigger a Close() flush failure.
- **Steps**: Run any export command (e.g., `export_ledgers`) with `--output` pointing to a file on a write-limited mount, OR use the `ulimit -f 0` technique from success/007's PoC. After Close() fails, verify the command exits 0 and check whether MaybeUpload would proceed.
- **Assertion**: Assert that the command exits with code 0 despite the Close() failure. Assert that the output file is empty or truncated. If testing upload path, assert MaybeUpload is invoked (or would be invoked if cloud flags were set) after the failed Close().
