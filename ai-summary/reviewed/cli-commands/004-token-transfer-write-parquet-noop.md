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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (sibling finding 003 covers `export_ledger_transaction`; this covers a different command)

### Trace Summary

`export_token_transfers` registers `--write-parquet` (via `AddCommonFlags` at init line 68) and `--parquet-output` (via `AddArchiveFlags` at init line 69). At runtime, `MustCommonFlags` parses `WriteParquet` into `commonArgs` (line 19), and `MustArchiveFlags` parses and returns the parquet path (line 21). However, the command discards the parquet path with `_` on line 21 (`startNum, path, _, limit := ...`) and never reads `commonArgs.WriteParquet`. The entire Run function (lines 17-63) contains no parquet accumulation, no `WriteParquet()` call, and no error when parquet is requested. Sibling commands like `export_contract_events` (structurally the closest — also processes per-ledger event arrays) correctly capture `parquetPath`, accumulate records when `WriteParquet` is true, and call `WriteParquet()`.

### Code Paths Examined

- `cmd/export_token_transfers.go:21` — `startNum, path, _, limit := utils.MustArchiveFlags(...)` discards parquet path
- `cmd/export_token_transfers.go:17-63` — entire Run function has no `WriteParquet` check or call
- `cmd/export_token_transfers.go:66-71` — `init()` registers both `AddCommonFlags` and `AddArchiveFlags`, advertising parquet flags
- `internal/utils/main.go:AddCommonFlags:245` — registers `--write-parquet` bool flag
- `internal/utils/main.go:AddArchiveFlags:253` — registers `--parquet-output` with default `exported_token_transfer.parquet`
- `internal/utils/main.go:MustArchiveFlags:541-563` — parses and returns `parquetPath` (line 552)
- `internal/utils/main.go:MustCommonFlags:519` — parses `WriteParquet` bool into `CommonFlags`
- `cmd/export_contract_events.go:19,50-52,63-66` — sibling event command correctly captures parquetPath, accumulates parquet records, and writes parquet
- `internal/transform/parquet_converter.go` — no `ToParquet()` method on `TokenTransferOutput`
- `internal/transform/schema_parquet.go` — no `TokenTransferOutputParquet` struct exists

### Findings

The bug has two layers:

1. **Silent no-op**: The command accepts `--write-parquet` and `--parquet-output` without error but produces no parquet output. An operator deploying this in a pipeline expecting parquet will get a successful exit code with no parquet file — a silent failure.

2. **Missing parquet infrastructure**: Even if the command tried to call `WriteParquet`, it would fail because `TokenTransferOutput` doesn't implement the `SchemaParquet` interface (no `ToParquet()` method) and there's no `TokenTransferOutputParquet` struct. The fix requires both command-level wiring AND a parquet schema/converter — or, at minimum, the command should reject the flags with an explicit error.

This is the same class of bug as H003 (`export_ledger_transaction`) but affects a different command. Of the 10 archive export commands, 7 correctly implement parquet support. `export_token_transfers` and `export_ledger_transaction` are the two outliers that silently drop parquet while advertising it through their CLI flags.

### PoC Guidance

- **Test file**: `cmd/export_token_transfers_test.go` (or append to an existing command test file)
- **Setup**: Register the `export_token_transfer` command with `--write-parquet --parquet-output /tmp/test_tt.parquet` flags and a small valid ledger range containing token transfers
- **Steps**: Execute the command with `--write-parquet` set. After successful completion, check whether the parquet output file exists at the specified path.
- **Assertion**: Assert that either (a) the parquet file exists and contains the exported token-transfer rows, or (b) the command returns a non-nil error / non-zero exit code when parquet is requested but unsupported. Currently neither condition holds — the command succeeds silently without producing parquet output.
