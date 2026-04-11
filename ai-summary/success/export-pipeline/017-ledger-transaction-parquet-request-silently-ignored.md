# 017: Ledger-transaction parquet request silently ignored

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`export_ledger_transaction` advertises and accepts `--write-parquet` plus `--parquet-output`, but a real CLI run exits successfully after writing only the JSON export. The requested parquet artifact is never created, so downstream parquet consumers can see a false-success export and silently miss the dataset they asked for.

This is a concrete runtime behavior, not just a static code smell. Running the production binary over a normal non-empty ledger range produced non-empty JSON output at `--output`, returned exit code 0, and still left the requested parquet path absent.

## Root Cause

The command parses the shared parquet flags but discards the returned `parquetPath` and never checks `commonArgs.WriteParquet`. Its run path always transforms transactions, writes JSON rows, closes the JSON file, prints success-style stats, and optionally uploads only the JSON path. There is also no ledger-transaction parquet schema wired into the transform layer, so there is no implementation path that could satisfy the advertised parquet request.

## Reproduction

Run `export_ledger_transaction --write-parquet --parquet-output <path>` over any non-empty ledger range. During normal execution the command fetches transactions from the configured archive backend, transforms them, writes JSON rows to `--output`, and exits successfully without ever branching into parquet generation. The requested parquet file remains missing even though the CLI accepted the parquet flags.

## Affected Code

- `internal/utils/main.go:AddCommonFlags:231-246` — registers `--write-parquet`
- `internal/utils/main.go:AddArchiveFlags:248-255` — registers `--parquet-output`
- `cmd/export_ledger_transaction.go:Run:17-21` — parses flags but discards `parquetPath`
- `cmd/export_ledger_transaction.go:Run:33-56` — writes JSON output, reports success, and never checks `commonArgs.WriteParquet` or calls `WriteParquet`
- `internal/transform/schema.go:LedgerTransactionOutput:86-94` — JSON output exists, but no parquet companion type is defined

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestExportLedgerTransactionParquetFlagIgnored`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, build the CLI with `go build -o stellar-etl .`, then run `go test ./cmd/... -run TestExportLedgerTransactionParquetFlagIgnored -v`.

### Test Body

```go
package cmd

import (
	"context"
	"os"
	"os/exec"
	"path/filepath"
	"runtime"
	"testing"
	"time"
)

func repoRoot(t *testing.T) string {
	t.Helper()

	_, filename, _, ok := runtime.Caller(0)
	if !ok {
		t.Fatal("could not resolve test file path")
	}

	return filepath.Dir(filepath.Dir(filename))
}

func TestExportLedgerTransactionParquetFlagIgnored(t *testing.T) {
	root := repoRoot(t)
	binaryPath := filepath.Join(root, "stellar-etl")
	if _, err := os.Stat(binaryPath); err != nil {
		t.Fatalf("expected built CLI binary at %s: %v", binaryPath, err)
	}

	tempDir := t.TempDir()
	jsonPath := filepath.Join(tempDir, "ledger-transactions.txt")
	parquetPath := filepath.Join(tempDir, "ledger-transactions.parquet")

	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Minute)
	defer cancel()

	cmd := exec.CommandContext(
		ctx,
		binaryPath,
		"export_ledger_transaction",
		"-s", "30820015",
		"-e", "30820015",
		"-o", jsonPath,
		"--write-parquet",
		"--parquet-output", parquetPath,
	)

	output, err := cmd.CombinedOutput()
	if err != nil {
		t.Fatalf("export_ledger_transaction failed: %v\n%s", err, string(output))
	}

	jsonInfo, err := os.Stat(jsonPath)
	if err != nil {
		t.Fatalf("expected JSON output at %s: %v\n%s", jsonPath, err, string(output))
	}
	if jsonInfo.Size() == 0 {
		t.Fatalf("expected non-empty JSON output at %s", jsonPath)
	}

	if _, err := os.Stat(parquetPath); !os.IsNotExist(err) {
		if err == nil {
			t.Fatalf("expected parquet output to be missing, but file exists at %s", parquetPath)
		}
		t.Fatalf("unexpected parquet stat result for %s: %v", parquetPath, err)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: when `--write-parquet` is requested, `export_ledger_transaction` should either emit a parquet file at `--parquet-output` or fail immediately with a clear unsupported-format error.
- **Actual**: the command exits successfully, writes non-empty JSON output, and leaves the requested parquet path missing.

## Adversarial Review

1. Exercises claimed bug: YES — the final PoC runs the real CLI against a normal ledger range and checks the actual filesystem outputs it produced.
2. Realistic preconditions: YES — the PoC uses only public command-line flags on a production command path with an ordinary non-empty ledger range.
3. Bug vs by-design: BUG — the command advertises parquet flags through shared helpers and never warns or errors that parquet is unsupported here.
4. Final severity: Medium — this silently drops a requested export artifact and reports success, but it does not alter numeric field values inside produced rows.
5. In scope: YES — this is a concrete export-pipeline correctness bug that can leave parquet-based downstream jobs with missing data.
6. Test correctness: CORRECT — the original source-only PoC was strengthened into an end-to-end runtime check that proves success status, real JSON output, and missing parquet output together.
7. Alternative explanations: NONE — the missing parquet file is explained by the fact that the command never uses the parsed parquet path or write-parquet flag.
8. Novelty: NOT ASSESSED HERE — duplicate handling belongs to the orchestrator.

## Suggested Fix

Either implement ledger-transaction parquet support end-to-end (schema, converter, `WriteParquet` call, and optional parquet upload) or reject `--write-parquet` for this command with a fatal unsupported-format error until support exists. Keep a regression test that proves the command cannot exit 0 while dropping a requested parquet file.
