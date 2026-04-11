# H004: `export_token_transfer` silently ignores parquet flags

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `export_token_transfer` accepts `--write-parquet` and `--parquet-output`, the command should write a parquet file for the same token-transfer rows it emits as JSON, or it should reject the unsupported flag combination. A successful run should not pretend parquet export happened when only JSON was produced.

## Mechanism

The command inherits the shared parquet flags from `AddCommonFlags` and `AddArchiveFlags`, but it discards the parsed parquet path and never checks `commonArgs.WriteParquet`. As a result, token-transfer exports always stop after JSON output, and callers requesting parquet get a silent no-op instead of the expected alternate format.

## Trigger

Run `export_token_transfer --write-parquet --parquet-output /tmp/token_transfers.parquet` over any ledger range containing token transfers. The command will finish successfully without creating or uploading the parquet file.

## Target Code

- `internal/utils/main.go:AddCommonFlags:232-245` — shared CLI surface advertises `--write-parquet`
- `internal/utils/main.go:AddArchiveFlags:250-254` — shared CLI surface advertises `--parquet-output`
- `internal/utils/main.go:MustArchiveFlags:541-563` — the parquet path is parsed for this command too
- `cmd/export_token_transfers.go:17-63` — the command discards `parquetPath` and never calls `WriteParquet`
- `internal/transform/schema.go:659-677` — token-transfer output exists only on the JSON path today

## Evidence

`export_token_transfer` uses `startNum, path, _, limit := utils.MustArchiveFlags(...)`, and the rest of the function has no parquet branch even though sibling archive exporters append transformed rows and invoke `WriteParquet`. There is likewise no `TokenTransferOutputParquet` schema or `TokenTransferOutput.ToParquet()` implementation in the transform package.

## Anti-Evidence

This does not affect ordinary JSON-only runs. The bug is the silent mismatch between the advertised flag surface and the actual export behavior when parquet is requested.
