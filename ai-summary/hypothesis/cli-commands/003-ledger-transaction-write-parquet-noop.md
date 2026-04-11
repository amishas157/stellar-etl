# H003: `export_ledger_transaction` silently ignores parquet flags

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `export_ledger_transaction` accepts `--write-parquet` and `--parquet-output`, the command should either emit a parquet file containing the same ledger-transaction rows as the JSON export or fail immediately and say parquet is unsupported. A successful run should not silently drop the requested output format.

## Mechanism

The shared flag helpers register parquet output flags for this command, and `MustArchiveFlags` returns a parquet path, but `export_ledger_transaction` discards that return value with `_` and never branches on `commonArgs.WriteParquet`. The command therefore completes normally after writing only JSON, leaving operators with no parquet file even though the CLI advertised and accepted the parquet options.

## Trigger

Run `export_ledger_transaction --write-parquet --parquet-output /tmp/ledger_transactions.parquet` over any non-empty ledger range. The command will exit successfully after producing only the JSON file at `--output`.

## Target Code

- `internal/utils/main.go:AddCommonFlags:232-245` — all archive commands advertise `--write-parquet`
- `internal/utils/main.go:AddArchiveFlags:250-254` — all archive commands advertise `--parquet-output`
- `internal/utils/main.go:MustArchiveFlags:541-563` — parquet path is parsed and returned to callers
- `cmd/export_ledger_transaction.go:17-57` — the command discards `parquetPath` and never calls `WriteParquet`
- `internal/transform/schema.go:86-94` — ledger-transaction output exists only on the JSON path today

## Evidence

Unlike the sibling exporters, this command binds `startNum, path, _, limit := utils.MustArchiveFlags(...)` and has no `if commonArgs.WriteParquet { ... }` block at all. There is also no ledger-transaction parquet schema or converter in `internal/transform/schema_parquet.go` / `parquet_converter.go`, which matches the observed no-op behavior.

## Anti-Evidence

JSON export still works correctly when parquet flags are omitted. This is a format-parity bug specific to runs that request parquet output.
