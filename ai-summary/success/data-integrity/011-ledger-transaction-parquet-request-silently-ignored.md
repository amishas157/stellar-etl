# 011: Ledger transaction parquet request is silently ignored

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`export_ledger_transaction` advertises and accepts the shared `--write-parquet` and `--parquet-output` flags, but a real CLI run never creates the requested parquet artifact. The command exits successfully and writes JSON output only, so downstream automation can treat the export as successful while silently missing the parquet deliverable.

## Root Cause

The command parses the shared parquet flags but discards the returned parquet path and never branches on `commonArgs.WriteParquet`. Unlike the sibling transaction exporter, it never accumulates parquet rows and never calls `WriteParquet()`, so parquet mode is unreachable even though the CLI surface exposes it.

## Reproduction

Run the ledger-transaction exporter with a normal ledger range and both parquet flags enabled. The command exits 0, creates a non-empty JSON file, and leaves the requested parquet path absent; running the sibling `export_transactions` command over the same ledger does create a parquet artifact, confirming the failure is specific to `export_ledger_transaction`.

## Affected Code

- `cmd/export_ledger_transaction.go:17-56` — parses archive flags, writes JSON rows, and finishes without any parquet branch or parquet upload.
- `internal/utils/main.go:232-255` — registers `--write-parquet` and `--parquet-output` for archive commands, including `export_ledger_transaction`.
- `internal/utils/main.go:519-563` — parses `WriteParquet` and `parquet-output`, but this command discards the parsed parquet path.
- `cmd/export_transactions.go:17-66` — sibling control path showing the expected `WriteParquet` flow when parquet support is actually wired.

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestLedgerTransactionParquetFlagAcceptedButNoop`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then run `go test ./cmd/... -run TestLedgerTransactionParquetFlagAcceptedButNoop -v`.

### Test Body

```go
package cmd

import (
	"os"
	"os/exec"
	"path/filepath"
	"strings"
	"testing"
)

func repoRoot(t *testing.T) string {
	t.Helper()

	dir, err := os.Getwd()
	if err != nil {
		t.Fatalf("get working directory: %v", err)
	}

	for {
		if _, err := os.Stat(filepath.Join(dir, "go.mod")); err == nil {
			return dir
		}

		parent := filepath.Dir(dir)
		if parent == dir {
			t.Fatal("could not locate repository root")
		}
		dir = parent
	}
}

func buildTestBinary(t *testing.T, root string) string {
	t.Helper()

	binaryPath := filepath.Join(t.TempDir(), "stellar-etl")
	cmd := exec.Command("go", "build", "-o", binaryPath, ".")
	cmd.Dir = root

	output, err := cmd.CombinedOutput()
	if err != nil {
		t.Fatalf("build stellar-etl binary: %v\n%s", err, output)
	}

	return binaryPath
}

func runExportCommand(t *testing.T, binaryPath, exportCommand, jsonPath, parquetPath string) string {
	t.Helper()

	cmd := exec.Command(
		binaryPath,
		exportCommand,
		"-s", "30820015",
		"-e", "30820015",
		"-o", jsonPath,
		"--write-parquet",
		"--parquet-output", parquetPath,
	)

	output, err := cmd.CombinedOutput()
	if err != nil {
		t.Fatalf("%s failed: %v\n%s", exportCommand, err, output)
	}

	return string(output)
}

func TestLedgerTransactionParquetFlagAcceptedButNoop(t *testing.T) {
	root := repoRoot(t)
	binaryPath := buildTestBinary(t, root)
	tempDir := t.TempDir()

	ledgerJSONPath := filepath.Join(tempDir, "ledger_transactions.json")
	ledgerParquetPath := filepath.Join(tempDir, "ledger_transactions.parquet")
	ledgerOutput := runExportCommand(t, binaryPath, "export_ledger_transaction", ledgerJSONPath, ledgerParquetPath)

	ledgerJSONInfo, err := os.Stat(ledgerJSONPath)
	if err != nil {
		t.Fatalf("ledger transaction export should create JSON output: %v", err)
	}
	if ledgerJSONInfo.Size() == 0 {
		t.Fatal("ledger transaction JSON output should not be empty")
	}
	if _, err := os.Stat(ledgerParquetPath); !os.IsNotExist(err) {
		t.Fatalf("ledger transaction export should not create parquet output, got err=%v", err)
	}
	if !strings.Contains(ledgerOutput, "successful_transforms") {
		t.Fatalf("ledger transaction export should report successful completion, output=%q", ledgerOutput)
	}

	transactionsJSONPath := filepath.Join(tempDir, "transactions.json")
	transactionsParquetPath := filepath.Join(tempDir, "transactions.parquet")
	transactionsOutput := runExportCommand(t, binaryPath, "export_transactions", transactionsJSONPath, transactionsParquetPath)

	transactionsJSONInfo, err := os.Stat(transactionsJSONPath)
	if err != nil {
		t.Fatalf("transactions export should create JSON output: %v", err)
	}
	if transactionsJSONInfo.Size() == 0 {
		t.Fatal("transactions JSON output should not be empty")
	}

	transactionsParquetInfo, err := os.Stat(transactionsParquetPath)
	if err != nil {
		t.Fatalf("transactions export should create parquet output: %v", err)
	}
	if transactionsParquetInfo.Size() == 0 {
		t.Fatal("transactions parquet output should not be empty")
	}
	if !strings.Contains(transactionsOutput, "successful_transforms") {
		t.Fatalf("transactions export should report successful completion, output=%q", transactionsOutput)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `export_ledger_transaction --write-parquet --parquet-output <path>` should either create a parquet artifact at `<path>` or fail explicitly.
- **Actual**: the command exits 0, writes JSON output, and leaves `<path>` absent.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC runs the real CLI command with the public parquet flags and inspects the real output files after successful completion.
2. Realistic preconditions: YES — any normal operator can invoke this command exactly this way; no mocks, private APIs, or impossible state are required.
3. Bug vs by-design: BUG — the CLI advertises a supported parquet mode, sibling exporters implement it, and this command silently accepts the request instead of rejecting unsupported usage.
4. Final severity: Medium — the bug silently drops an expected export artifact while reporting success; that is operational data loss rather than direct numeric corruption.
5. In scope: YES — this is a concrete data-export correctness failure on a public CLI path.
6. Test correctness: CORRECT — the assertion checks observable runtime behavior and uses `export_transactions` as a control to rule out a generic parquet or environment failure.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Either implement ledger-transaction parquet export end-to-end (`LedgerTransactionOutputParquet`, `ToParquet()`, accumulation, `WriteParquet()`, and upload handling), or reject `--write-parquet` for this command with an explicit error instead of silently succeeding.
