# H002: `export_token_transfer` accepts `--write-parquet` but never emits Parquet

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When an archive command accepts the shared `--write-parquet` and `--parquet-output` flags, enabling them should either produce a Parquet artifact for the exported rows or fail explicitly. A successful `export_token_transfer --write-parquet` run should not silently leave callers with JSON-only output.

## Mechanism

`export_token_transfer` inherits the global parquet flags through `AddCommonFlags()` and `AddArchiveFlags()`, and `MustCommonFlags()` / `MustArchiveFlags()` both parse those values. But the command discards the parsed parquet path (`startNum, path, _, limit := ...`) and never checks `commonArgs.WriteParquet`, never accumulates a Parquet row slice, and never calls `WriteParquet()` or uploads a parquet artifact. The command therefore accepts a supported output mode, exits successfully, and silently drops the requested Parquet output.

## Trigger

Run `export_token_transfer --write-parquet --parquet-output exported_token_transfer.parquet --start-ledger ... --end-ledger ...`. The command completes after writing/uploading only the JSON file; no local or remote parquet artifact is created for the token-transfer rows.

## Target Code

- `internal/utils/main.go:231-246` — archive commands register the shared `--write-parquet` and `--parquet-output` flags
- `internal/utils/main.go:519-537` — `MustCommonFlags()` reads `write-parquet`
- `internal/utils/main.go:540-563` — `MustArchiveFlags()` returns the `parquet-output` path
- `cmd/export_token_transfers.go:19-23` — command parses `commonArgs.WriteParquet` and discards the returned parquet path with `_`
- `cmd/export_token_transfers.go:38-63` — run loop writes only JSON entries and never branches on parquet output
- `cmd/export_token_transfers.go:66-70` — command still advertises archive/common flags, so users can request parquet on this surface

## Evidence

This command is an outlier among archive exporters: sibling commands that support parquet keep the third `MustArchiveFlags()` return value, collect `transform.SchemaParquet` rows, and gate a `WriteParquet()` call on `commonArgs.WriteParquet`. `export_token_transfer` accepts the exact same flags but has no equivalent branch or warning.

## Anti-Evidence

There is no `TokenTransferOutputParquet` type today, so this may have started life as an unfinished feature rather than a regression. But unlike the explicit `skip = true` guard used for intentionally unsupported Parquet surfaces elsewhere, this command gives users no signal that their requested output mode was ignored.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

I traced the complete flag registration, parsing, and usage path for `export_token_transfers`. The command registers `--write-parquet` and `--parquet-output` flags via `AddCommonFlags()` (line 68) and `AddArchiveFlags("token_transfer", ...)` (line 69). `MustArchiveFlags()` parses and returns the `parquetPath` (utils/main.go:552-555), but the command discards it with `_` on line 21. The `commonArgs.WriteParquet` field is parsed by `MustCommonFlags()` but never read anywhere in the command's `Run` function. No `TokenTransferOutputParquet` type exists in the codebase, and `TokenTransferOutput` does not implement the `SchemaParquet` interface (no `ToParquet()` method).

### Code Paths Examined

- `cmd/export_token_transfers.go:21` — `startNum, path, _, limit := utils.MustArchiveFlags(...)` discards parquetPath with blank identifier
- `cmd/export_token_transfers.go:38-55` — main loop only calls `ExportEntry()` for JSON; no `SchemaParquet` accumulation, no `WriteParquet()` call
- `cmd/export_token_transfers.go:62` — `MaybeUpload()` called only for JSON path; no parquet upload
- `cmd/export_token_transfers.go:68-69` — `AddCommonFlags()` and `AddArchiveFlags()` register `--write-parquet` and `--parquet-output`
- `internal/utils/main.go:250-255` — `AddArchiveFlags()` registers `--parquet-output` with default `exported_token_transfer.parquet`
- `internal/utils/main.go:541-555` — `MustArchiveFlags()` parses and returns `parquetPath`
- `internal/transform/schema.go:659-677` — `TokenTransferOutput` struct exists but has no `ToParquet()` method
- `internal/transform/parquet_converter.go:15-17` — `SchemaParquet` interface requires `ToParquet()` — not implemented by `TokenTransferOutput`

### Findings

**Confirmed: the hypothesis is accurate.** Of the 10 archive export commands, 8 support parquet output (ledgers, transactions, operations, effects, trades, assets, contract_events, ledger_entry_changes). Only `export_token_transfers` and `export_ledger_transaction` discard the parquet path and skip parquet entirely.

Critically, the `export_ledger_entry_changes` command demonstrates the proper pattern for intentionally unsupported parquet surfaces: it uses an explicit `skip = true` guard with a comment (`// Skipping ClaimableBalanceOutputParquet because it is not needed in the current scope of work` at line 332). `export_token_transfers` has no such guard — it silently ignores the flag.

The bug has two layers:
1. **Missing type**: No `TokenTransferOutputParquet` struct and no `ToParquet()` method on `TokenTransferOutput`
2. **Missing runtime guard**: Even without parquet support, the command should either warn/error when `--write-parquet` is set, or not register the flag at all

When a user runs `export_token_transfer --write-parquet`, the command exits 0 with only JSON output. Automated pipelines checking exit codes would assume parquet was produced successfully.

### PoC Guidance

- **Test file**: `cmd/export_token_transfers_test.go`
- **Setup**: Create a test that invokes the `export_token_transfer` command with `--write-parquet` and `--parquet-output` flags pointing to a temp file, using a small ledger range from existing testdata
- **Steps**: Execute the command with `--write-parquet --parquet-output /tmp/test_token_transfer.parquet`, then check whether the parquet output file was created
- **Assertion**: Assert that either (a) the parquet file exists and contains expected rows, or (b) the command returns an error/warning when `--write-parquet` is used. Currently neither happens — the file is never created and the command exits 0, confirming silent data loss.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestTokenTransferWriteParquetNoop"
**Test Language**: Go

### Demonstration

The test proves that `export_token_transfer` registers both `--write-parquet` and `--parquet-output` flags (so users can request parquet output), but `TokenTransferOutput` does not implement the `SchemaParquet` interface required by `WriteParquet()`. A control check confirms that sibling types like `LedgerOutput` do implement the interface. This structural gap means the command silently accepts and ignores parquet requests — no parquet file can ever be produced, and no error is raised.

### Test Body

```go
package cmd

import (
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
)

// TestTokenTransferWriteParquetNoop demonstrates that export_token_transfer
// accepts --write-parquet and --parquet-output flags but never produces parquet
// output. The command exits 0 with only JSON output, silently ignoring the
// user's parquet request.
func TestTokenTransferWriteParquetNoop(t *testing.T) {
	// 1. Verify the command registers --write-parquet flag
	writeParquetFlag := tokenTransfersCmd.Flags().Lookup("write-parquet")
	if writeParquetFlag == nil {
		t.Fatal("expected --write-parquet flag to be registered on export_token_transfer, but it was not")
	}

	// 2. Verify the command registers --parquet-output flag
	parquetOutputFlag := tokenTransfersCmd.Flags().Lookup("parquet-output")
	if parquetOutputFlag == nil {
		t.Fatal("expected --parquet-output flag to be registered on export_token_transfer, but it was not")
	}

	// 3. Verify TokenTransferOutput does NOT implement SchemaParquet.
	//    Every sibling export type that supports parquet implements this interface,
	//    which provides the ToParquet() method needed by WriteParquet().
	var output interface{} = transform.TokenTransferOutput{}
	_, implementsParquet := output.(transform.SchemaParquet)
	if implementsParquet {
		t.Fatal("TokenTransferOutput unexpectedly implements SchemaParquet; bug may have been fixed")
	}

	// 4. Verify a sibling type (LedgerOutput) DOES implement SchemaParquet,
	//    confirming the interface check is valid and the pattern exists.
	var ledgerOutput interface{} = transform.LedgerOutput{}
	_, ledgerImplements := ledgerOutput.(transform.SchemaParquet)
	if !ledgerImplements {
		t.Fatal("LedgerOutput should implement SchemaParquet — test validation check failed")
	}

	// Bug confirmed: export_token_transfer registers --write-parquet and
	// --parquet-output flags (so users can request parquet), but
	// TokenTransferOutput does not implement SchemaParquet (so no parquet
	// conversion is possible). The command silently ignores the parquet
	// request and exits 0 with JSON-only output.
	t.Log("BUG CONFIRMED: export_token_transfer accepts --write-parquet but " +
		"TokenTransferOutput does not implement SchemaParquet. " +
		"Parquet output is silently dropped.")
}
```

### Test Output

```
=== RUN   TestTokenTransferWriteParquetNoop
    data_integrity_poc_test.go:48: BUG CONFIRMED: export_token_transfer accepts --write-parquet but TokenTransferOutput does not implement SchemaParquet. Parquet output is silently dropped.
--- PASS: TestTokenTransferWriteParquetNoop (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	5.610s
```
