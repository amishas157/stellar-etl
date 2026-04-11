# 010: Token transfer parquet request is silently ignored

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`export_token_transfer` advertises and accepts the shared `--write-parquet` and `--parquet-output` flags, but it never generates a parquet artifact. A real CLI run succeeds, writes JSON output, and leaves the requested parquet path missing, so automation can treat the export as successful while silently losing the parquet deliverable.

## Root Cause

The command parses the shared parquet flags but discards the returned parquet path, never branches on `commonArgs.WriteParquet`, never accumulates parquet rows, and never calls `WriteParquet()`. Token transfer exports also lack a parquet conversion path, so the requested mode is unreachable even though the CLI surface exposes it.

## Reproduction

Run the token transfer exporter with a normal ledger range and both parquet flags enabled. The command exits successfully and produces a non-empty JSON file, but no parquet file is created at the user-supplied `--parquet-output` path.

## Affected Code

- `cmd/export_token_transfers.go:tokenTransfersCmd.Run:17-62` — parses archive flags, writes JSON rows, and finishes without any parquet branch or parquet upload.
- `internal/utils/main.go:AddCommonFlags/AddArchiveFlags/MustCommonFlags/MustArchiveFlags:231-255,458-563` — registers and parses `--write-parquet` and `--parquet-output` for this command.
- `cmd/command_utils.go:WriteParquet:148-180` — parquet emission requires `[]transform.SchemaParquet`, but token transfer export never reaches this path.
- `internal/transform/schema.go:TokenTransferOutput:659-677` — token transfer has only the JSON output shape wired here; no parquet output path is defined alongside the command.

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestTokenTransferWriteParquetNoop`
- **Test language**: `go`
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

func TestTokenTransferWriteParquetNoop(t *testing.T) {
	wd, err := os.Getwd()
	if err != nil {
		t.Fatalf("getwd: %v", err)
	}

	repoRoot := wd
	if filepath.Base(wd) == "cmd" {
		repoRoot = filepath.Dir(wd)
	}

	tempDir := t.TempDir()
	jsonPath := filepath.Join(tempDir, "token_transfers.json")
	parquetPath := filepath.Join(tempDir, "token_transfers.parquet")

	cmd := exec.Command(
		"go",
		"run",
		".",
		"export_token_transfer",
		"-s", "30820015",
		"-e", "30820015",
		"-o", jsonPath,
		"--write-parquet",
		"--parquet-output", parquetPath,
	)
	cmd.Dir = repoRoot
	output, err := cmd.CombinedOutput()
	if err != nil {
		t.Fatalf("command failed: %v\n%s", err, output)
	}

	jsonInfo, err := os.Stat(jsonPath)
	if err != nil {
		t.Fatalf("expected JSON output file to exist: %v\n%s", err, output)
	}
	if jsonInfo.Size() == 0 {
		t.Fatalf("expected JSON output file to be non-empty\n%s", output)
	}

	if _, err := os.Stat(parquetPath); err == nil {
		t.Fatalf("expected parquet output file to be absent because the command silently ignores --write-parquet; file exists at %s", parquetPath)
	} else if !os.IsNotExist(err) {
		t.Fatalf("unexpected parquet output stat error: %v", err)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `export_token_transfer --write-parquet --parquet-output <path>` should either create a parquet artifact at `<path>` or fail explicitly.
- **Actual**: the command exits 0, writes JSON output, and leaves `<path>` absent.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC runs the real CLI command with the public parquet flags and observes successful execution without the requested parquet artifact.
2. Realistic preconditions: YES — any normal operator can invoke this command exactly this way; no internal APIs or mocks are required.
3. Bug vs by-design: BUG — unsupported parquet surfaces elsewhere are handled explicitly, but this command advertises the mode and silently accepts it.
4. Final severity: Medium — the bug drops an expected export artifact without surfacing an error, which is silent operational data loss rather than numeric corruption.
5. In scope: YES — this is a concrete export correctness failure on a public data-export path.
6. Test correctness: CORRECT — the assertion checks the real output files after a successful command run and fails if parquet is actually produced.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Either implement token transfer parquet export end-to-end (`TokenTransferOutputParquet`, `ToParquet()`, accumulation, `WriteParquet()`, and upload ordering), or reject `--write-parquet` for this command with an explicit error instead of silently succeeding.
