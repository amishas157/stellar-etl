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

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestParquetFlagsSilentlyIgnoredByTokenTransferAndLedgerTransaction"
**Test Language**: Go

### Demonstration

The test proves that both `export_token_transfer` and `export_ledger_transaction` cobra commands register `--write-parquet` and `--parquet-output` flags (verified via flag lookup on the command objects), but their source code discards the parquet path with `_` and never references `WriteParquet` or `commonArgs.WriteParquet`. In contrast, the sibling `export_assets` command properly captures `parquetPath`, checks `commonArgs.WriteParquet`, and calls `WriteParquet()`. This confirms users can pass the parquet flags without error but receive no parquet output and no warning.

### Test Body

```go
package cmd

import (
	"os"
	"path/filepath"
	"runtime"
	"strings"
	"testing"
)

// TestParquetFlagsSilentlyIgnoredByTokenTransferAndLedgerTransaction demonstrates
// that export_token_transfer and export_ledger_transaction register --write-parquet
// and --parquet-output flags (via AddCommonFlags/AddArchiveFlags) but silently
// discard them. Users who pass these flags get a successful exit with no parquet
// file and no warning.
func TestParquetFlagsSilentlyIgnoredByTokenTransferAndLedgerTransaction(t *testing.T) {
	// Part 1: Verify both commands DO register the parquet flags.
	// This proves users can pass --write-parquet without a CLI error.

	// Locate the cmd/ source directory relative to this test file
	_, thisFile, _, _ := runtime.Caller(0)
	cmdDir := filepath.Dir(thisFile)

	cmdsUnderTest := map[string]*struct {
		sourceFile string
	}{
		"export_token_transfer":     {sourceFile: filepath.Join(cmdDir, "export_token_transfers.go")},
		"export_ledger_transaction": {sourceFile: filepath.Join(cmdDir, "export_ledger_transaction.go")},
	}

	for cmdName, meta := range cmdsUnderTest {
		// Find the command in the root command tree
		cmd, _, err := rootCmd.Find([]string{cmdName})
		if err != nil {
			t.Fatalf("command %q not found: %v", cmdName, err)
		}

		// Verify --write-parquet flag is registered
		wpFlag := cmd.Flags().Lookup("write-parquet")
		if wpFlag == nil {
			t.Fatalf("%s: --write-parquet flag not registered (expected it to be registered but ignored)", cmdName)
		}

		// Verify --parquet-output flag is registered
		poFlag := cmd.Flags().Lookup("parquet-output")
		if poFlag == nil {
			t.Fatalf("%s: --parquet-output flag not registered (expected it to be registered but ignored)", cmdName)
		}

		t.Logf("%s: --write-parquet and --parquet-output flags ARE registered (users can pass them)", cmdName)

		// Part 2: Read the source file and verify the parquet path is discarded
		// and WriteParquet is never called.
		src, err := os.ReadFile(meta.sourceFile)
		if err != nil {
			t.Fatalf("could not read source file %s: %v", meta.sourceFile, err)
		}
		source := string(src)

		// Check that parquetPath is assigned to blank identifier `_`
		// The pattern in both files is: startNum, path, _, limit := utils.MustArchiveFlags(...)
		if !strings.Contains(source, "_, limit := utils.MustArchiveFlags") {
			t.Errorf("%s: expected parquetPath to be discarded with `_` in MustArchiveFlags destructuring", cmdName)
		} else {
			t.Logf("%s: CONFIRMED parquetPath is discarded with `_` in MustArchiveFlags call", cmdName)
		}

		// Check that WriteParquet is never called
		if strings.Contains(source, "WriteParquet") {
			t.Errorf("%s: unexpectedly found WriteParquet call (hypothesis would be wrong)", cmdName)
		} else {
			t.Logf("%s: CONFIRMED WriteParquet is never called", cmdName)
		}

		// Check that commonArgs.WriteParquet is never referenced
		if strings.Contains(source, "commonArgs.WriteParquet") {
			t.Errorf("%s: unexpectedly found commonArgs.WriteParquet reference", cmdName)
		} else {
			t.Logf("%s: CONFIRMED commonArgs.WriteParquet is never checked", cmdName)
		}
	}

	// Part 3: Verify a sibling command (export_assets) DOES properly use parquet flags.
	// This proves the pattern exists and the two commands are outliers.
	assetsSrc, err := os.ReadFile(filepath.Join(cmdDir, "export_assets.go"))
	if err != nil {
		t.Fatalf("could not read export_assets.go: %v", err)
	}
	assetsSource := string(assetsSrc)

	if !strings.Contains(assetsSource, "parquetPath, limit := utils.MustArchiveFlags") {
		t.Errorf("export_assets: expected parquetPath to be captured (not discarded)")
	} else {
		t.Logf("export_assets: CONFIRMED parquetPath IS captured (correct behavior)")
	}

	if !strings.Contains(assetsSource, "commonArgs.WriteParquet") {
		t.Errorf("export_assets: expected commonArgs.WriteParquet to be checked")
	} else {
		t.Logf("export_assets: CONFIRMED commonArgs.WriteParquet IS checked (correct behavior)")
	}

	if !strings.Contains(assetsSource, "WriteParquet(") {
		t.Errorf("export_assets: expected WriteParquet() to be called")
	} else {
		t.Logf("export_assets: CONFIRMED WriteParquet() IS called (correct behavior)")
	}

	t.Log("\n=== SUMMARY ===")
	t.Log("Both export_token_transfer and export_ledger_transaction register --write-parquet")
	t.Log("and --parquet-output flags, but silently discard the parquet path and never check")
	t.Log("WriteParquet. A user passing --write-parquet gets NO parquet output and NO warning.")
	t.Log("In contrast, export_assets (and 6 other sibling commands) properly use both flags.")
}
```

### Test Output

```
=== RUN   TestParquetFlagsSilentlyIgnoredByTokenTransferAndLedgerTransaction
    data_integrity_poc_test.go:49: export_token_transfer: --write-parquet and --parquet-output flags ARE registered (users can pass them)
    data_integrity_poc_test.go:64: export_token_transfer: CONFIRMED parquetPath is discarded with `_` in MustArchiveFlags call
    data_integrity_poc_test.go:71: export_token_transfer: CONFIRMED WriteParquet is never called
    data_integrity_poc_test.go:78: export_token_transfer: CONFIRMED commonArgs.WriteParquet is never checked
    data_integrity_poc_test.go:49: export_ledger_transaction: --write-parquet and --parquet-output flags ARE registered (users can pass them)
    data_integrity_poc_test.go:64: export_ledger_transaction: CONFIRMED parquetPath is discarded with `_` in MustArchiveFlags call
    data_integrity_poc_test.go:71: export_ledger_transaction: CONFIRMED WriteParquet is never called
    data_integrity_poc_test.go:78: export_ledger_transaction: CONFIRMED commonArgs.WriteParquet is never checked
    data_integrity_poc_test.go:93: export_assets: CONFIRMED parquetPath IS captured (correct behavior)
    data_integrity_poc_test.go:99: export_assets: CONFIRMED commonArgs.WriteParquet IS checked (correct behavior)
    data_integrity_poc_test.go:105: export_assets: CONFIRMED WriteParquet() IS called (correct behavior)
    data_integrity_poc_test.go:108:
        === SUMMARY ===
    data_integrity_poc_test.go:109: Both export_token_transfer and export_ledger_transaction register --write-parquet
    data_integrity_poc_test.go:110: and --parquet-output flags, but silently discard the parquet path and never check
    data_integrity_poc_test.go:111: WriteParquet. A user passing --write-parquet gets NO parquet output and NO warning.
    data_integrity_poc_test.go:112: In contrast, export_assets (and 6 other sibling commands) properly use both flags.
--- PASS: TestParquetFlagsSilentlyIgnoredByTokenTransferAndLedgerTransaction (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	1.868s
```
