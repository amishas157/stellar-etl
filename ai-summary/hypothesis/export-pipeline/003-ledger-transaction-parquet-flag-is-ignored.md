# H003: `export_ledger_transaction --write-parquet` is silently ignored

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: Requested parquet output silently missing
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a command exposes `--write-parquet` and `--parquet-output`, enabling those flags should either create the requested parquet file or fail fast with an explicit unsupported-output error. `export_ledger_transaction` should not accept the flags and then produce only JSON.

## Mechanism

`export_ledger_transaction` inherits archive flags, and `MustArchiveFlags()` reads a parquet path for it, but the command never accumulates parquet records and never calls `WriteParquet()` or uploads `parquet-output`. The CLI therefore accepts a legitimate user request for parquet output and exits successfully after writing only the text export, silently dropping the requested dataset variant.

## Trigger

Run `stellar-etl export_ledger_transaction --start-ledger <n> --end-ledger <m> --write-parquet --parquet-output ledger_transaction.parquet`.

## Target Code

- `cmd/export_ledger_transaction.go:17-57` — command writes JSON rows and uploads only `path`
- `cmd/export_ledger_transaction.go:60-65` — command registers archive flags, so parquet flags are exposed to the user
- `internal/utils/main.go:250-255` — `AddArchiveFlags()` defines `--parquet-output`
- `internal/utils/main.go:541-563` — `MustArchiveFlags()` reads `parquet-output` even for this command

## Evidence

Sibling archive commands such as `export_transactions`, `export_ledgers`, and `export_contract_events` all gate a `WriteParquet(...)` call on `commonArgs.WriteParquet`. `export_ledger_transaction` is the outlier: it parses the parquet path but never references it again, so the command surface promises an output mode that the implementation does not honor.

## Anti-Evidence

There is no `LedgerTransactionOutputParquet` type today, so this may be an unfinished feature rather than a regression. But the current behavior is still a concrete user-visible correctness bug because the command accepts the flag and reports success without producing the requested parquet artifact.
