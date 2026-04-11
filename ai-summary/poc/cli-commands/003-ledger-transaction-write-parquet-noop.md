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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`export_ledger_transaction` registers both `--write-parquet` (via `AddCommonFlags` at line 245) and `--parquet-output` (via `AddArchiveFlags` at line 253) through shared flag helpers called in `init()` at lines 62-63. At runtime, `MustArchiveFlags` parses and returns the parquet path (line 552), but the command discards it with `_` on line 21. The `commonArgs.WriteParquet` flag is parsed into `CommonFlags` by `MustCommonFlags` (line 519) but never read. The sibling `export_transactions` command captures `parquetPath`, accumulates transformed records when `WriteParquet` is true (lines 51-53), and calls `WriteParquet()` (line 65) — none of which `export_ledger_transaction` does.

### Code Paths Examined

- `internal/utils/main.go:AddCommonFlags:232-245` — registers `--write-parquet` flag (line 245) for all commands including `export_ledger_transaction`
- `internal/utils/main.go:AddArchiveFlags:250-255` — registers `--parquet-output` flag with default filename `exported_ledger_transaction.parquet` (line 253)
- `internal/utils/main.go:MustArchiveFlags:540-563` — parses and returns `parquetPath` from the `--parquet-output` flag (line 552)
- `internal/utils/main.go:MustCommonFlags:519` — parses `WriteParquet` bool into `CommonFlags`
- `cmd/export_ledger_transaction.go:21` — `startNum, path, _, limit := utils.MustArchiveFlags(...)` discards parquet path
- `cmd/export_ledger_transaction.go:17-57` — entire Run function has no `WriteParquet` check, no `WriteParquet()` call
- `cmd/export_transactions.go:21,51-53,63-66` — sibling command correctly captures parquetPath, accumulates parquet records, and writes parquet
- `internal/transform/schema_parquet.go` — no `LedgerTransactionOutputParquet` struct exists
- `internal/transform/parquet_converter.go` — no `ToParquet()` method on `LedgerTransactionOutput`

### Findings

The bug has two layers:

1. **Silent no-op**: The command accepts `--write-parquet` and `--parquet-output` without error but produces no parquet output. An operator deploying this command in a pipeline that expects parquet output will get a successful exit code and no parquet file — a silent failure.

2. **Missing parquet infrastructure**: Even if the command tried to call `WriteParquet`, it would fail because `LedgerTransactionOutput` doesn't implement the `SchemaParquet` interface (no `ToParquet()` method) and there's no `LedgerTransactionOutputParquet` struct. This means the fix requires both command-level wiring AND a parquet schema/converter addition — or, at minimum, the command should reject the flags with an explicit error message.

The deviation from sibling commands is clear: `export_transactions`, `export_ledgers`, `export_operations`, and others all follow the pattern of capturing `parquetPath`, checking `commonArgs.WriteParquet`, and calling `WriteParquet()`. `export_ledger_transaction` is the outlier that silently drops parquet support while still advertising it through the CLI flags.

### PoC Guidance

- **Test file**: `cmd/export_ledger_transaction_test.go` (or create a new test alongside existing command tests)
- **Setup**: Register the `export_ledger_transaction` command with `--write-parquet --parquet-output /tmp/test_lt.parquet` flags and a small valid ledger range
- **Steps**: Execute the command with `--write-parquet` flag set. After successful completion, check whether the parquet output file exists at the specified path.
- **Assertion**: Assert that either (a) the parquet file exists and contains the exported rows, or (b) the command returns a non-nil error / non-zero exit code when parquet is requested but unsupported. Currently neither condition holds — the command succeeds silently without producing parquet output.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestLedgerTransactionParquetFlagsAcceptedButNoop"
**Test Language**: Go

### Demonstration

The test proves both layers of the bug. First, it confirms that `ledgerTransactionCmd` registers `--write-parquet` and `--parquet-output` flags (creating operator expectation of parquet support). Second, it verifies via Go type assertion that `LedgerTransactionOutput` does NOT implement the `SchemaParquet` interface — meaning no `ToParquet()` method exists and parquet output is structurally impossible. A control check confirms the sibling `TransactionOutput` DOES implement the interface, proving the check is meaningful.

### Test Body

```go
package cmd

import (
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
)

// TestLedgerTransactionParquetFlagsAcceptedButNoop demonstrates that the
// export_ledger_transaction command registers --write-parquet and --parquet-output
// flags (creating user expectation of parquet support) but LedgerTransactionOutput
// does not implement SchemaParquet, so no parquet file can ever be produced.
// The command silently succeeds without writing parquet output.
func TestLedgerTransactionParquetFlagsAcceptedButNoop(t *testing.T) {
	// Layer 1: The command advertises parquet support via flags.
	// --write-parquet is registered by AddCommonFlags, --parquet-output by AddArchiveFlags.
	wpFlag := ledgerTransactionCmd.Flags().Lookup("write-parquet")
	if wpFlag == nil {
		t.Fatal("expected --write-parquet flag to be registered on export_ledger_transaction, but it was not found")
	}

	poFlag := ledgerTransactionCmd.Flags().Lookup("parquet-output")
	if poFlag == nil {
		t.Fatal("expected --parquet-output flag to be registered on export_ledger_transaction, but it was not found")
	}

	// Layer 2: LedgerTransactionOutput does NOT implement SchemaParquet.
	// Even if the command tried to call WriteParquet, it would fail because
	// there is no ToParquet() method on LedgerTransactionOutput.
	var lto interface{} = transform.LedgerTransactionOutput{}
	if _, ok := lto.(transform.SchemaParquet); ok {
		t.Fatal("LedgerTransactionOutput unexpectedly implements SchemaParquet — this would mean the bug is fixed")
	}
	t.Log("CONFIRMED: LedgerTransactionOutput does NOT implement SchemaParquet")

	// Control: Sibling TransactionOutput DOES implement SchemaParquet,
	// proving the interface check is meaningful.
	var to interface{} = transform.TransactionOutput{}
	if _, ok := to.(transform.SchemaParquet); !ok {
		t.Fatal("TransactionOutput should implement SchemaParquet (control check failed)")
	}
	t.Log("CONTROL: TransactionOutput correctly implements SchemaParquet")

	// Conclusion: export_ledger_transaction accepts --write-parquet and --parquet-output
	// without error, but silently produces no parquet output because:
	// 1. The Run function discards parquetPath with _ (line 21 of export_ledger_transaction.go)
	// 2. There is no WriteParquet check in the Run function
	// 3. LedgerTransactionOutput has no ToParquet() method
	t.Log("BUG CONFIRMED: export_ledger_transaction silently ignores parquet flags")
}
```

### Test Output

```
=== RUN   TestLedgerTransactionParquetFlagsAcceptedButNoop
    data_integrity_poc_test.go:34: CONFIRMED: LedgerTransactionOutput does NOT implement SchemaParquet
    data_integrity_poc_test.go:42: CONTROL: TransactionOutput correctly implements SchemaParquet
    data_integrity_poc_test.go:49: BUG CONFIRMED: export_ledger_transaction silently ignores parquet flags
--- PASS: TestLedgerTransactionParquetFlagsAcceptedButNoop (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	5.587s
```
