# H002: `export_token_transfer` accepts `--write-parquet` but never emits Parquet

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When an archive command accepts the shared `--write-parquet` and `--parquet-output` flags, enabling them should either produce a Parquet artifact for the exported rows or fail explicitly. A successful `export_token_transfer --write-parquet` run should not silently leave callers with JSON-only output.

## Mechanism

`export_token_transfer` inherits the global parquet flags through `AddCommonFlags()` and `AddArchiveFlags()`, and `MustCommonFlags()` / `MustArchiveFlags()` both parse those values. But the command discards the parsed parquet path (`startNum, path, _, limit := ...`) and never checks `commonArgs.WriteParquet`, never accumulates a Parquet row slice, and never calls `WriteParquet()` or uploads a parquet artifact. The command therefore accepts a supported output mode, exits successfully, and silently drops the requested Parquet output.

## Trigger

Run `export_token_transfer --write-parquet --parquet-output exported_token_transfer.parquet --start-ledger ... --end-ledger ...`. The command completes after writing/uploading only the JSON file; no local or remote parquet artifact is created for the token-transfer rows.

## Target Code

- `internal/utils/main.go:231-246` — archive commands register the shared `--write-parquet` and `--parquet-output` flags
- `internal/utils/main.go:519-537` — `MustCommonFlags()` reads `write-parquet`
- `internal/utils/main.go:540-563` — `MustArchiveFlags()` returns the `parquet-output` path
- `cmd/export_token_transfers.go:19-23` — command parses `commonArgs.WriteParquet` and discards the returned parquet path with `_`
- `cmd/export_token_transfers.go:38-63` — run loop writes only JSON entries and never branches on parquet output
- `cmd/export_token_transfers.go:66-70` — command still advertises archive/common flags, so users can request parquet on this surface

## Evidence

This command is an outlier among archive exporters: sibling commands that support parquet keep the third `MustArchiveFlags()` return value, collect `transform.SchemaParquet` rows, and gate a `WriteParquet()` call on `commonArgs.WriteParquet`. `export_token_transfer` accepts the exact same flags but has no equivalent branch or warning.

## Anti-Evidence

There is no `TokenTransferOutputParquet` type today, so this may have started life as an unfinished feature rather than a regression. But unlike the explicit `skip = true` guard used for intentionally unsupported Parquet surfaces elsewhere, this command gives users no signal that their requested output mode was ignored.
