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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Both `export_token_transfers.go` and `export_ledger_transaction.go` register `--write-parquet` (via `AddCommonFlags`) and `--parquet-output` (via `AddArchiveFlags`) but discard the parquet path with `_` in their `MustArchiveFlags` destructuring and never check `commonArgs.WriteParquet`. This is confirmed by comparison with 7 sibling archive commands (ledgers, transactions, operations, effects, trades, assets, contract_events) that all properly use `parquetPath` and branch on `WriteParquet`. Additionally, neither `TokenTransferOutput` nor `LedgerTransactionOutput` implements the `SchemaParquet` interface (no `ToParquet()` method), confirming that Parquet support was never wired in for these types.

### Code Paths Examined

- `internal/utils/main.go:245` — `AddCommonFlags` registers `--write-parquet` flag for all commands
- `internal/utils/main.go:250-254` — `AddArchiveFlags` registers `--parquet-output` with default filename
- `internal/utils/main.go:541` — `MustArchiveFlags` returns `parquetPath` as third return value
- `cmd/export_token_transfers.go:21` — destructures as `startNum, path, _, limit` — parquetPath discarded
- `cmd/export_token_transfers.go:36-62` — no `WriteParquet` call, no `commonArgs.WriteParquet` check
- `cmd/export_ledger_transaction.go:21` — destructures as `startNum, path, _, limit` — parquetPath discarded
- `cmd/export_ledger_transaction.go:30-56` — no `WriteParquet` call, no `commonArgs.WriteParquet` check
- `cmd/export_assets.go:21,67,79-81` — sibling correctly uses `parquetPath`, checks `WriteParquet`, calls `WriteParquet()`
- `internal/transform/parquet_converter.go` — no `ToParquet()` method exists for `TokenTransferOutput` or `LedgerTransactionOutput`

### Findings

The bug is confirmed at two levels:

1. **Command level**: Both commands accept `--write-parquet` and `--parquet-output` flags without error but silently ignore them. A user or automation script running `export_token_transfer --write-parquet` gets a successful exit with no parquet file and no warning. This violates the consistent flag contract established by all 7 other archive commands.

2. **Schema level**: Neither `TokenTransferOutput` nor `LedgerTransactionOutput` implements the `SchemaParquet` interface (no `ToParquet()` method, no corresponding `*Parquet` struct in `schema_parquet.go`). This means the fix requires both adding Parquet schemas AND wiring the command-level plumbing.

The severity is correctly Medium — this is an operational correctness issue (silent flag acceptance without action) rather than data corruption. The JSON output path is correct for both commands.

### PoC Guidance

- **Test file**: `cmd/export_token_transfers_test.go` and `cmd/export_ledger_transaction_test.go` (or a new integration test)
- **Setup**: Configure a small ledger range with known token transfer / ledger transaction data
- **Steps**: Run the command with `--write-parquet --parquet-output /tmp/test.parquet` and check if the parquet file exists after command completion
- **Assertion**: Assert that the parquet file either (a) does not exist (proving the flag is ignored) or (b) exists but is empty/zero-bytes. Additionally, assert that no warning or error is logged about parquet being unsupported. This confirms the silent-ignore behavior.
