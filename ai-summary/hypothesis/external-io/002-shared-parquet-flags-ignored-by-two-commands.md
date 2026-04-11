# H002: Shared parquet flags are silently ignored by `export_token_transfer` and `export_ledger_transaction`

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a user sets `--write-parquet` and `--parquet-output` on an archive export command, the command should create the requested Parquet artifact (and upload it if cloud output is enabled), matching the shared flag contract used by sibling exporters.

## Mechanism

Both commands inherit the shared `write-parquet` and `parquet-output` flags, but their run functions discard the parsed parquet path and never branch on `commonArgs.WriteParquet`. The command completes successfully with only JSON output, so callers who requested Parquet receive no file and no warning that the flag was ignored.

## Trigger

Run either command with `--write-parquet --parquet-output some-file.parquet`, for example:

- `stellar-etl export_token_transfer --start-ledger ... --end-ledger ... --write-parquet`
- `stellar-etl export_ledger_transaction --start-ledger ... --end-ledger ... --write-parquet`

Both runs accept the flags and exit normally without creating the requested Parquet file.

## Target Code

- `internal/utils/main.go:245-255` — registers shared `write-parquet` and `parquet-output` flags for archive commands
- `internal/utils/main.go:519-537` — parses `write-parquet` into `CommonFlagValues`
- `internal/utils/main.go:540-563` — parses `parquet-output` in `MustArchiveFlags`
- `cmd/export_token_transfers.go:17-63` — discards `parquetPath` as `_` and never checks `commonArgs.WriteParquet`
- `cmd/export_ledger_transaction.go:17-57` — discards `parquetPath` as `_` and never checks `commonArgs.WriteParquet`

## Evidence

Both commands destructure `MustArchiveFlags` as `startNum, path, _, limit := ...`, proving the parquet path is intentionally thrown away. Unlike sibling commands such as `export_assets`, `export_effects`, and `export_contract_events`, neither command accumulates `SchemaParquet` rows or calls `WriteParquet()` / `MaybeUpload()` for Parquet output.

## Anti-Evidence

The JSON export path still works correctly, so users who only want line-delimited JSON are unaffected. The issue only appears when callers rely on the shared Parquet flags behaving consistently across archive commands.
