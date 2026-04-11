# 016: Token-transfer parquet request silently ignored

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`export_token_transfer` advertises `--write-parquet` and `--parquet-output`, accepts both flags, and still exits successfully without producing the requested parquet artifact. The command writes a normal JSON export, reports success, and never signals that the parquet request was ignored, which can leave downstream parquet pipelines with silent data gaps.

## Root Cause

The command parses the shared parquet flags, but `export_token_transfers.go` discards the parsed parquet path and never branches on `commonArgs.WriteParquet`. Its run path always emits JSON, closes the JSON file, prints success stats, and only considers the JSON output path for upload.

## Reproduction

Run the real CLI against a ledger range already used by the repository's token-transfer fixtures with both parquet flags enabled. The command returns exit code 0, writes non-empty JSON token-transfer output, and leaves the requested parquet path absent.

## Affected Code

- `internal/utils/main.go:AddCommonFlags:231-246` — registers `--write-parquet` on all commands
- `internal/utils/main.go:AddArchiveFlags:248-255` — registers `--parquet-output` on archive exports
- `internal/utils/main.go:MustCommonFlags:458-537` — parses `write-parquet`
- `internal/utils/main.go:MustArchiveFlags:540-560` — parses `parquet-output`
- `cmd/export_token_transfers.go:Run:17-63` — discards `parquetPath`, writes JSON only, and never calls `WriteParquet`

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestTokenTransferParquetFlagsIgnored`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package cmd

import (
	"bytes"
	"os"
	"os/exec"
	"path/filepath"
	"runtime"
	"testing"
)

func TestTokenTransferParquetFlagsIgnored(t *testing.T) {
	if tokenTransfersCmd.Flags().Lookup("write-parquet") == nil {
		t.Fatal("export_token_transfer does not register --write-parquet")
	}
	if tokenTransfersCmd.Flags().Lookup("parquet-output") == nil {
		t.Fatal("export_token_transfer does not register --parquet-output")
	}

	_, thisFile, _, ok := runtime.Caller(0)
	if !ok {
		t.Fatal("resolve current test file path")
	}
	repoRoot := filepath.Dir(filepath.Dir(thisFile))

	tmpDir := t.TempDir()
	binaryPath := filepath.Join(tmpDir, "stellar-etl")
	buildCmd := exec.Command("go", "build", "-o", binaryPath, ".")
	buildCmd.Dir = repoRoot
	buildOutput, err := buildCmd.CombinedOutput()
	if err != nil {
		t.Fatalf("build stellar-etl: %v\n%s", err, buildOutput)
	}

	jsonPath := filepath.Join(tmpDir, "token-transfers.json")
	parquetPath := filepath.Join(tmpDir, "token-transfers.parquet")
	exportCmd := exec.Command(
		binaryPath,
		"export_token_transfer",
		"-s", "30820015",
		"-e", "30820015",
		"-o", jsonPath,
		"--write-parquet",
		"--parquet-output", parquetPath,
	)
	exportCmd.Dir = repoRoot
	exportOutput, err := exportCmd.CombinedOutput()
	if err != nil {
		t.Fatalf("run export_token_transfer: %v\n%s", err, exportOutput)
	}

	jsonBytes, err := os.ReadFile(jsonPath)
	if err != nil {
		t.Fatalf("read JSON export: %v\n%s", err, exportOutput)
	}
	if len(bytes.TrimSpace(jsonBytes)) == 0 {
		t.Fatalf("expected non-empty JSON export\n%s", exportOutput)
	}

	if _, err := os.Stat(parquetPath); err == nil {
		t.Fatalf("expected parquet output to be missing despite --write-parquet, but %s exists", parquetPath)
	} else if !os.IsNotExist(err) {
		t.Fatalf("stat parquet output: %v", err)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `export_token_transfer --write-parquet --parquet-output <path>` either creates a parquet artifact containing the requested token-transfer rows or fails loudly that parquet export is unsupported.
- **Actual**: The command succeeds, writes JSON rows, and leaves the requested parquet output path missing.

## Adversarial Review

1. Exercises claimed bug: YES — the test builds the real CLI, runs `export_token_transfer` with the parquet flags on a real token-transfer ledger, and checks the actual filesystem result.
2. Realistic preconditions: YES — the PoC uses only shipped CLI flags and normal command execution.
3. Bug vs by-design: BUG — the command advertises and parses the parquet flags but silently ignores them instead of rejecting unsupported behavior.
4. Final severity: Medium — this causes silent export gaps and false-success runs rather than wrong numeric field values.
5. In scope: YES — this matches the allowed medium-severity class for silent export failure in the data pipeline.
6. Test correctness: CORRECT — it proves the command both succeeds and produces real JSON output before asserting the requested parquet artifact is absent.
7. Alternative explanations: NONE — if parquet were intentionally unsupported here, the command should reject the request rather than silently accept it.
8. Novelty: NOT ASSESSED HERE — duplicate handling is owned by the orchestrator.

## Suggested Fix

Either implement token-transfer parquet export end-to-end (schema, converter, `WriteParquet` call, and parquet upload) or reject `--write-parquet` for this command with a fatal error until support exists.
