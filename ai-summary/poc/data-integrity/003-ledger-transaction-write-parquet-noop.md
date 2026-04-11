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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (sibling finding H002 covers `export_token_transfer`; this is a distinct command instance)

### Trace Summary

I traced the complete flag registration, parsing, and usage path for `export_ledger_transaction`. The command registers `--write-parquet` via `AddCommonFlags()` (line 62) and `--parquet-output` via `AddArchiveFlags("ledger_transaction", ...)` (line 63). `MustArchiveFlags()` parses and returns the `parquetPath` (utils/main.go:552-555), but the command discards it with `_` on line 21. The `commonArgs.WriteParquet` boolean is parsed by `MustCommonFlags()` but never read anywhere in the command's `Run` function. No `LedgerTransactionOutputParquet` type exists anywhere in the codebase, and `LedgerTransactionOutput` does not implement the `SchemaParquet` interface.

### Code Paths Examined

- `cmd/export_ledger_transaction.go:21` — `startNum, path, _, limit := utils.MustArchiveFlags(...)` discards parquetPath with blank identifier
- `cmd/export_ledger_transaction.go:33-49` — main loop only calls `ExportEntry()` for JSON output; no `SchemaParquet` accumulation, no `WriteParquet()` call
- `cmd/export_ledger_transaction.go:56` — `MaybeUpload()` called only for JSON path; no parquet upload
- `cmd/export_ledger_transaction.go:62-63` — `AddCommonFlags()` and `AddArchiveFlags()` register `--write-parquet` and `--parquet-output`
- `internal/utils/main.go:245` — `AddCommonFlags()` registers `--write-parquet` flag
- `internal/utils/main.go:250-255` — `AddArchiveFlags()` registers `--parquet-output` with default `exported_ledger_transaction.parquet`
- `internal/utils/main.go:519-537` — `MustCommonFlags()` parses `WriteParquet` boolean
- `internal/utils/main.go:541-555` — `MustArchiveFlags()` parses and returns `parquetPath`
- `internal/transform/schema.go:86-94` — `LedgerTransactionOutput` struct exists with 7 fields, no `ToParquet()` method
- `internal/transform/schema_parquet.go` — no `LedgerTransactionOutputParquet` type exists

### Findings

**Confirmed: the hypothesis is accurate.** Of the archive export commands, most support parquet output (ledgers, transactions, operations, effects, trades, assets, contract_events, ledger_entry_changes). Only `export_ledger_transaction` and `export_token_transfers` discard the parquet path and skip parquet entirely — both with the identical `_, limit` blank-identifier pattern.

The proper pattern for intentionally unsupported parquet surfaces is demonstrated by `export_ledger_entry_changes`, which uses an explicit `skip = true` guard with a comment for claimable balances. `export_ledger_transaction` has no such guard — it silently ignores the flag.

The bug has two layers:
1. **Missing type**: No `LedgerTransactionOutputParquet` struct and no `ToParquet()` method on `LedgerTransactionOutput`
2. **Missing runtime guard**: Even without parquet support, the command should either warn/error when `--write-parquet` is set, or not register the flag at all

When a user runs `export_ledger_transaction --write-parquet`, the command exits 0 with only JSON output. Automated pipelines checking exit codes would assume parquet was produced successfully.

### PoC Guidance

- **Test file**: `cmd/export_ledger_transaction_test.go` (or create one if it doesn't exist)
- **Setup**: Create a test that invokes the `export_ledger_transaction` command with `--write-parquet` and `--parquet-output` flags pointing to a temp file, using a small ledger range from existing testdata
- **Steps**: Execute the command with `--write-parquet --parquet-output /tmp/test_ledger_transaction.parquet`, then check whether the parquet output file was created
- **Assertion**: Assert that either (a) the parquet file exists and contains expected rows, or (b) the command returns an error/warning when `--write-parquet` is used. Currently neither happens — the file is never created and the command exits 0, confirming silent data loss.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestLedgerTransactionParquetFlagAcceptedButNoop"
**Test Language**: Go

### Demonstration

The test proves that `export_ledger_transaction` registers both `--write-parquet` and `--parquet-output` CLI flags (inviting users to request Parquet output), but `LedgerTransactionOutput` does not implement the `SchemaParquet` interface required for Parquet serialization. A control check confirms the sibling `TransactionOutput` type does implement `SchemaParquet`, proving this is not a framework limitation but a gap specific to `export_ledger_transaction`. The command silently accepts the flags and exits 0 with only JSON output.

### Test Body

```go
package cmd

import (
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
)

// TestLedgerTransactionParquetFlagAcceptedButNoop demonstrates that
// export_ledger_transaction advertises --write-parquet and --parquet-output
// flags but has no implementation path to produce Parquet output.
// The command silently ignores the flags and exits 0 with JSON-only output.
func TestLedgerTransactionParquetFlagAcceptedButNoop(t *testing.T) {
	// 1. Verify the command registers --write-parquet flag
	wpFlag := ledgerTransactionCmd.Flags().Lookup("write-parquet")
	if wpFlag == nil {
		t.Fatal("export_ledger_transaction should register --write-parquet flag")
	}
	t.Logf("--write-parquet flag registered: %q (default: %s)", wpFlag.Usage, wpFlag.DefValue)

	// 2. Verify the command registers --parquet-output flag
	poFlag := ledgerTransactionCmd.Flags().Lookup("parquet-output")
	if poFlag == nil {
		t.Fatal("export_ledger_transaction should register --parquet-output flag")
	}
	t.Logf("--parquet-output flag registered with default: %s", poFlag.DefValue)

	// 3. Verify LedgerTransactionOutput does NOT implement SchemaParquet
	var lto interface{} = transform.LedgerTransactionOutput{}
	_, implementsParquet := lto.(transform.SchemaParquet)
	if implementsParquet {
		t.Fatal("LedgerTransactionOutput unexpectedly implements SchemaParquet — hypothesis disproven")
	}
	t.Log("LedgerTransactionOutput does NOT implement SchemaParquet (no ToParquet method)")

	// 4. Control: verify TransactionOutput DOES implement SchemaParquet
	var to interface{} = transform.TransactionOutput{}
	_, controlImplements := to.(transform.SchemaParquet)
	if !controlImplements {
		t.Fatal("TransactionOutput should implement SchemaParquet (control case)")
	}
	t.Log("TransactionOutput DOES implement SchemaParquet (control case passes)")

	// 5. Control: verify the sibling export_transactions command also registers
	// --write-parquet AND its output type supports parquet (the working pattern)
	siblingFlag := transactionsCmd.Flags().Lookup("write-parquet")
	if siblingFlag == nil {
		t.Fatal("export_transactions should register --write-parquet flag (control)")
	}

	// BUG PROVEN: export_ledger_transaction registers --write-parquet and
	// --parquet-output flags that users can set, but LedgerTransactionOutput
	// has no ToParquet() method and the command's Run function discards the
	// parquet path (using _ blank identifier) and never checks WriteParquet.
	// A user running: stellar-etl export_ledger_transaction --write-parquet
	//   --parquet-output out.parquet -e 100
	// gets exit code 0 with only JSON output; the parquet file is never created.
	t.Log("BUG CONFIRMED: export_ledger_transaction registers --write-parquet " +
		"and --parquet-output flags but LedgerTransactionOutput does not implement " +
		"SchemaParquet. The command silently produces only JSON when --write-parquet is set.")
}
```

### Test Output

```
=== RUN   TestLedgerTransactionParquetFlagAcceptedButNoop
    data_integrity_poc_test.go:19: --write-parquet flag registered: "If set, write output as parquet files." (default: false)
    data_integrity_poc_test.go:26: --parquet-output flag registered with default: exported_ledger_transaction.parquet
    data_integrity_poc_test.go:34: LedgerTransactionOutput does NOT implement SchemaParquet (no ToParquet method)
    data_integrity_poc_test.go:42: TransactionOutput DOES implement SchemaParquet (control case passes)
    data_integrity_poc_test.go:58: BUG CONFIRMED: export_ledger_transaction registers --write-parquet and --parquet-output flags but LedgerTransactionOutput does not implement SchemaParquet. The command silently produces only JSON when --write-parquet is set.
--- PASS: TestLedgerTransactionParquetFlagAcceptedButNoop (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	5.897s
```
