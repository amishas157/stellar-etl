# H003: `export_ledger_transaction` accepts `--write-parquet` but never emits Parquet

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `export_ledger_transaction` exposes the shared `--write-parquet` and `--parquet-output` flags, a successful run with those flags enabled should either materialize a Parquet file for `LedgerTransactionOutput` rows or reject the unsupported mode up front. It should not silently behave as JSON-only while claiming success.

## Mechanism

The command inherits and parses the parquet flags exactly like other archive exporters, but it immediately discards the parsed parquet path (`startNum, path, _, limit := ...`) and never checks `commonArgs.WriteParquet` again. Because the run loop only calls `ExportEntry()` on `LedgerTransactionOutput` values and then uploads the JSON path, the command completes without any local Parquet write, upload, or warning that the requested output format was ignored.

## Trigger

Run `export_ledger_transaction --write-parquet --parquet-output exported_ledger_transaction.parquet --start-ledger ... --end-ledger ...`. The command exits successfully after producing only the JSON artifact; the requested parquet file is never created or uploaded.

## Target Code

- `internal/utils/main.go:231-246` — archive commands register `--write-parquet` and `--parquet-output`
- `internal/utils/main.go:519-537` — `MustCommonFlags()` exposes `WriteParquet` to commands
- `internal/utils/main.go:540-563` — `MustArchiveFlags()` returns `parquetPath`
- `cmd/export_ledger_transaction.go:19-23` — command parses `commonArgs.WriteParquet` and discards `parquetPath`
- `cmd/export_ledger_transaction.go:33-57` — command writes JSON rows only and has no Parquet branch
- `cmd/export_ledger_transaction.go:60-64` — the init path still advertises the shared archive/common flag set to users

## Evidence

The implementation matches the same no-op pattern as `export_token_transfer`: parquet flags are accepted at the CLI layer but not connected to any writer path. Unlike exporters such as `export_assets` or `export_effects`, there is no `[]transform.SchemaParquet` accumulator, no `WriteParquet()` call, and no explicit unsupported-mode guard.

## Anti-Evidence

`LedgerTransactionOutput` is a raw-XDR-heavy export and may simply never have had Parquet support implemented. Even so, the current behavior still looks like a live CLI integrity bug because the command advertises and parses the shared parquet flags, then silently ignores them instead of failing fast.
