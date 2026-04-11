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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the `export_ledger_transaction` command from flag registration through execution. `AddArchiveFlags("ledger_transaction", ...)` at line 63 registers `--parquet-output` with default `exported_ledger_transaction.parquet`. `AddCommonFlags` registers `--write-parquet`. However, at line 21, the command explicitly discards the parquetPath return value from `MustArchiveFlags` with `_`, never checks `commonArgs.WriteParquet`, never accumulates a `[]transform.SchemaParquet` slice, and never calls `WriteParquet()`. The command exits 0 after writing only JSON.

### Code Paths Examined

- `cmd/export_ledger_transaction.go:21` — `startNum, path, _, limit := utils.MustArchiveFlags(...)` explicitly discards parquetPath with blank identifier
- `cmd/export_ledger_transaction.go:17-57` — entire Run func has no reference to `WriteParquet`, `parquetPath`, or `commonArgs.WriteParquet`
- `cmd/export_ledger_transaction.go:62-63` — `AddCommonFlags` + `AddArchiveFlags` register both `--write-parquet` and `--parquet-output` flags
- `cmd/export_transactions.go:21,33,51-53,63-65` — sibling command captures parquetPath, accumulates parquet records, and calls `WriteParquet()` when flag is set
- `internal/utils/main.go:245` — `AddCommonFlags` registers `--write-parquet` bool flag
- `internal/utils/main.go:250-255` — `AddArchiveFlags` registers `--parquet-output` string flag
- `internal/utils/main.go:541-563` — `MustArchiveFlags` parses and returns parquetPath as 3rd value
- `internal/transform/` — no `LedgerTransactionOutputParquet` type exists anywhere

### Findings

The bug is confirmed. `export_ledger_transaction` advertises `--write-parquet` and `--parquet-output` via shared flag helpers but never implements parquet output. The blank identifier `_` on line 21 is the smoking gun — the parquet path is intentionally discarded. No `LedgerTransactionOutputParquet` type exists in the transform package, so even if the flag were checked, there's no parquet schema to write to. This is the same class of bug as the confirmed success/016 finding for `export_token_transfer`.

### PoC Guidance

- **Test file**: `cmd/export_ledger_transaction_test.go` (create if not exists, or append to existing command test file)
- **Setup**: Configure a minimal ledger range with test fixtures (use existing testdata). Set `--write-parquet` and `--parquet-output /tmp/test_ledger_tx.parquet`.
- **Steps**: Execute `export_ledger_transaction` with both parquet flags enabled against a small ledger range. Verify command exits 0 with JSON output.
- **Assertion**: Assert that the parquet output file at the specified `--parquet-output` path either (a) does not exist, or (b) is empty/zero-length — demonstrating the flag was silently ignored despite the command reporting success.
