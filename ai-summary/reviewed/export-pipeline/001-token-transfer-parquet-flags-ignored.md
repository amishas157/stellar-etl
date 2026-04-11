# H001: Token-transfer export silently ignores parquet flags

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_token_transfer` is invoked with `--write-parquet` and `--parquet-output`, it should either write a parquet artifact containing the same token-transfer rows as the JSON export or fail immediately that parquet output is unsupported.

## Mechanism

The command registers and parses the global parquet flags, but then discards the parquet output path and never checks `commonArgs.WriteParquet`. The run path always writes JSON, closes the JSON file, and optionally uploads only that JSON path, so callers can request parquet output and still get a success exit with no parquet dataset.

## Trigger

Run `export_token_transfer --write-parquet --parquet-output /tmp/token_transfer.parquet -s <start> -e <end>` over any range that contains at least one SEP-41 token transfer event. The command completes successfully, writes JSON rows, and never creates the requested parquet artifact.

## Target Code

- `internal/utils/main.go:AddCommonFlags:231-246` — exposes `--write-parquet`
- `internal/utils/main.go:AddArchiveFlags:248-255` — exposes `--parquet-output`
- `internal/utils/main.go:MustCommonFlags:458-537` — reads `write-parquet`
- `cmd/export_token_transfers.go:tokenTransfersCmd.Run:17-63` — parses the parquet path into `_` and never calls `WriteParquet`

## Evidence

`export_token_transfer` calls `utils.MustCommonFlags()` and `utils.MustArchiveFlags()`, so the command accepts both parquet flags. But `cmd/export_token_transfers.go` binds the third archive return value to `_` and has no branch on `commonArgs.WriteParquet`; after the JSON loop it only calls `MaybeUpload(..., path)` for the JSON file.

## Anti-Evidence

The command help text still says it exports JSON objects, so the implementation may predate parquet support. But the CLI surface now advertises the parquet flags globally, which makes the silent omission user-visible and correctness-relevant.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`MustArchiveFlags` returns four values: `(startNum, path, parquetPath, limit)`. In `export_token_transfers.go:21`, the third return value (`parquetPath`) is explicitly discarded into `_`. The command body contains no `commonArgs.WriteParquet` check and no `WriteParquet()` call. Every other archive-based export command (ledgers, transactions, operations, effects, trades, assets, contract_events) captures `parquetPath` and gates parquet output on `commonArgs.WriteParquet`. Additionally, no `TokenTransferOutputParquet` struct exists in `schema_parquet.go`, confirming parquet conversion was never implemented for this entity.

### Code Paths Examined

- `internal/utils/main.go:MustArchiveFlags:541` — returns `parquetPath` as third value
- `cmd/export_token_transfers.go:21` — assigns third return value to `_`, discarding `parquetPath`
- `cmd/export_token_transfers.go:17-63` — no `WriteParquet` call, no `commonArgs.WriteParquet` branch
- `cmd/export_ledgers.go:21,58,70-72` — comparison: captures `parquetPath`, checks `WriteParquet`, calls `WriteParquet()`
- `cmd/export_transactions.go:21,51,63-65` — comparison: same parquet pattern as ledgers
- `cmd/export_operations.go:21,51,63-65` — comparison: same parquet pattern
- `cmd/export_effects.go:21,53,66-68` — comparison: same parquet pattern
- `cmd/export_trades.go:24,55,68-70` — comparison: same parquet pattern
- `cmd/export_assets.go:21,67,79-81` — comparison: same parquet pattern
- `internal/transform/schema_parquet.go` — no `TokenTransferOutputParquet` type exists
- `cmd/export_ledger_transaction.go:21` — also discards `parquetPath` with `_` (same pattern)

### Findings

The bug is confirmed at two levels:

1. **Command level**: `export_token_transfers.go` accepts `--write-parquet` and `--parquet-output` flags (via `AddCommonFlags` and `AddArchiveFlags` in `init()`), but the `Run` function discards the parquet path and never checks the write-parquet flag. The command exits with success code 0 and no parquet file.

2. **Schema level**: No `TokenTransferOutputParquet` struct exists in `schema_parquet.go`, and no `ConvertToTokenTransferParquet()` method exists in `parquet_converter.go`. So even if the command tried to call `WriteParquet`, there would be no Parquet schema to write with.

This is a pattern violation: 7 of 9 archive-based export commands implement the parquet path. The 2 outliers (`export_token_transfers` and `export_ledger_transaction`) silently accept and ignore the flags.

The impact is that automated pipelines requesting parquet output for token transfers will receive a success exit code but no parquet artifact, causing downstream data gaps or stale data consumption without any error signal.

### PoC Guidance

- **Test file**: `cmd/export_token_transfers_test.go` (create if not existing, or append to nearest test file)
- **Setup**: Configure a mock or testnet ledger range containing at least one SEP-41 token transfer event. Set `--write-parquet` and `--parquet-output /tmp/test_token_transfer.parquet`.
- **Steps**: Execute `export_token_transfer` with the parquet flags. Check the exit code and whether the parquet file at the specified path exists.
- **Assertion**: Assert that either (a) the parquet file exists and contains the same rows as the JSON output, OR (b) the command returns a non-zero exit code / logs an error indicating parquet is unsupported for this entity type. Currently, neither condition is met — the command exits 0 with no parquet file.
