# 010: `export_ledger_transaction` silently ignores requested Parquet output

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Subsystem**: cli-commands
**Final review by**: gpt-5.4, high

## Summary

`export_ledger_transaction` advertises and accepts `--write-parquet` plus `--parquet-output`, but a real CLI run exits successfully after writing only the JSON output. The requested parquet artifact is never created, so operators can believe a dual-format export succeeded when the parquet side was silently dropped.

This is not just a missing interface implementation in isolation. The production command path was reproduced end-to-end with a real ledger export: JSON was emitted, exit status was 0, and the requested parquet path remained absent.

## Root Cause

The command parses the shared parquet flags but discards the returned `parquetPath` and never checks `commonArgs.WriteParquet`. It always follows the JSON-only path and then optionally uploads only the JSON file. There is also no ledger-transaction parquet schema/converter wired into the transform layer, so the command has no implementation path that could satisfy the advertised parquet request.

## Reproduction

Run `export_ledger_transaction --write-parquet --parquet-output <path>` over any non-empty ledger range. During normal execution the command fetches transactions, transforms them, writes JSON rows to `--output`, closes that file, logs success-style stats, and exits 0 without ever branching into parquet generation. The requested parquet file is left missing even though the CLI accepted the parquet flags.

## Affected Code

- `internal/utils/main.go:232-245` — shared archive commands register `--write-parquet`
- `internal/utils/main.go:250-254` — shared archive commands register `--parquet-output`
- `cmd/export_ledger_transaction.go:17-21` — parses flags but discards `parquetPath`
- `cmd/export_ledger_transaction.go:33-56` — writes JSON output and uploads it, but never checks `commonArgs.WriteParquet` or calls `WriteParquet`
- `internal/transform/schema.go:86-94` — ledger-transaction output exists only as the JSON struct today

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestLedgerTransactionParquetFlagsAcceptedButNoop`
- **Test language**: go
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package cmd

import (
	"os"
	"os/exec"
	"path/filepath"
	"testing"
)

func findBuiltBinary(t *testing.T) string {
	t.Helper()

	wd, err := os.Getwd()
	if err != nil {
		t.Fatalf("get working directory: %v", err)
	}

	candidates := []string{
		filepath.Join(wd, "stellar-etl"),
		filepath.Join(filepath.Dir(wd), "stellar-etl"),
	}

	for _, candidate := range candidates {
		info, err := os.Stat(candidate)
		if err == nil && !info.IsDir() {
			return candidate
		}
	}

	t.Fatalf("could not find built stellar-etl binary from %s", wd)
	return ""
}

func TestLedgerTransactionParquetFlagsAcceptedButNoop(t *testing.T) {
	executable := findBuiltBinary(t)

	tempDir := t.TempDir()
	jsonPath := filepath.Join(tempDir, "ledger_transactions.json")
	parquetPath := filepath.Join(tempDir, "ledger_transactions.parquet")

	cmd := exec.Command(
		executable,
		"export_ledger_transaction",
		"-s", "30820015",
		"-e", "30820015",
		"-l", "1",
		"-o", jsonPath,
		"--write-parquet",
		"--parquet-output", parquetPath,
	)

	output, err := cmd.CombinedOutput()
	if err != nil {
		t.Fatalf("export_ledger_transaction returned an error: %v\noutput:\n%s", err, output)
	}

	jsonInfo, err := os.Stat(jsonPath)
	if err != nil {
		t.Fatalf("expected JSON output file to be created: %v\noutput:\n%s", err, output)
	}
	if jsonInfo.Size() == 0 {
		t.Fatalf("expected JSON output file to be non-empty\noutput:\n%s", output)
	}

	_, err = os.Stat(parquetPath)
	if err == nil {
		t.Fatalf("expected requested parquet output to be missing, but %s exists", parquetPath)
	}
	if !os.IsNotExist(err) {
		t.Fatalf("unexpected error checking parquet output: %v", err)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: when `--write-parquet` is requested, `export_ledger_transaction` should either emit a parquet file at `--parquet-output` or fail immediately with a clear unsupported-format error.
- **Actual**: the command exits successfully, writes only JSON, and leaves the requested parquet path missing.

## Adversarial Review

1. Exercises claimed bug: YES — the final PoC runs the production CLI command against a real ledger range and checks the actual filesystem outputs.
2. Realistic preconditions: YES — the flags are public CLI options on this command, and the tested ledger range is the same kind of normal export range already used by the command's own test coverage.
3. Bug vs by-design: BUG — the command advertises parquet flags through the shared helpers and gives no warning or error that parquet is unsupported.
4. Final severity: Medium — this does not corrupt row contents, but it silently drops a requested export artifact while reporting success.
5. In scope: YES — this is a concrete export-path correctness bug that can leave downstream parquet consumers with missing data.
6. Test correctness: CORRECT — the original static interface check was insufficient, so the final PoC was rewritten to verify the end-to-end command behavior and resulting files.
7. Alternative explanations: NONE — the requested parquet path is ignored because the command never uses it.
8. Novelty: NOVEL

## Suggested Fix

Either implement ledger-transaction parquet support end-to-end (schema, converter, command wiring) or reject `--write-parquet`/`--parquet-output` for this command with a fatal unsupported-format error before any export begins. Keep a regression test that proves the command cannot exit 0 while dropping the requested parquet file.
