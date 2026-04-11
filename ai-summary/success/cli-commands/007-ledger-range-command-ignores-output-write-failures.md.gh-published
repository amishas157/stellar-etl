# 007: Ledger Range Command Ignores Output Write Failures

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: cli-commands
**Final review by**: gpt-5.4, high

## Summary

`get_ledger_range_from_times` writes its JSON result directly to the requested `--output` file and ignores the return values from `Write`, `WriteString`, and `Close`. A real kernel-level write failure therefore leaves an empty or truncated range file on disk while the command still exits successfully, so downstream automation can consume a corrupt export boundary as if the lookup succeeded.

## Root Cause

The command bypasses the shared export helpers and performs raw file I/O inline. Its file-output branch never checks whether the JSON payload was fully written, whether the trailing newline was written, or whether buffered data failed to flush on `Close`.

## Reproduction

Run the real CLI with `-o <path>` under a process file-size limit of zero bytes. The command still returns exit code 0, but the requested output file is created as an empty file because the write failure is silently discarded.

## Affected Code

- `cmd/get_ledger_range_from_times.go:Run:74-78` — writes JSON bytes and newline directly to the output file and ignores `Write`, `WriteString`, and `Close` errors
- `cmd/command_utils.go:MustOutFile:31-52` — opens the writable file handle used by the command's unchecked write path

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestLedgerRangeWriteErrorsSilentlyIgnored`
- **Test language**: `go`
- **How to run**: `go build ./... && go test ./cmd/... -run '^TestLedgerRangeWriteErrorsSilentlyIgnored$' -v`

### Test Body

```go
package cmd

import (
	"os"
	"os/exec"
	"path/filepath"
	"runtime"
	"testing"
)

func TestLedgerRangeWriteErrorsSilentlyIgnored(t *testing.T) {
	_, thisFile, _, ok := runtime.Caller(0)
	if !ok {
		t.Fatal("could not determine test file path")
	}

	repoRoot := filepath.Dir(filepath.Dir(thisFile))
	binaryPath := filepath.Join(t.TempDir(), "stellar-etl")

	buildCmd := exec.Command("go", "build", "-o", binaryPath, ".")
	buildCmd.Dir = repoRoot
	buildOutput, err := buildCmd.CombinedOutput()
	if err != nil {
		t.Fatalf("could not build stellar-etl: %v\n%s", err, buildOutput)
	}

	outPath := filepath.Join(t.TempDir(), "ledger-range.json")
	runCmd := exec.Command("bash", "-lc", `ulimit -f 0 && "$BIN" get_ledger_range_from_times -s 2016-11-10T18:00:00-05:00 -e 2019-09-13T23:00:00+00:00 -o "$OUT"`)
	runCmd.Dir = repoRoot
	runCmd.Env = append(os.Environ(),
		"BIN="+binaryPath,
		"OUT="+outPath,
	)
	runOutput, err := runCmd.CombinedOutput()
	if err != nil {
		t.Fatalf("expected command to exit successfully despite write failure, got %v\n%s", err, runOutput)
	}

	content, err := os.ReadFile(outPath)
	if err != nil {
		t.Fatalf("could not read output file: %v", err)
	}

	if len(content) != 0 {
		t.Fatalf("expected empty output file after silent write failure, got %d bytes: %q", len(content), content)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: If the range JSON or trailing newline cannot be written, the command should return a non-zero error and refuse to leave a success-shaped output artifact behind.
- **Actual**: The command exits 0 after a real file write failure and leaves an empty or truncated output file at the requested `--output` path.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC builds the real `stellar-etl` binary and runs the actual `get_ledger_range_from_times -o ...` command path.
2. Realistic preconditions: YES — a zero-byte file-size limit is a deterministic stand-in for real post-open filesystem failures such as quota exhaustion, `ENOSPC`, or flush failures.
3. Bug vs by-design: BUG — a CLI export boundary command should not report success when its requested output file was not written.
4. Final severity: Medium — this is silent operational data loss that can mislead downstream automation into reading an empty or malformed range file.
5. In scope: YES — it affects correctness of supported CLI export workflows.
6. Test correctness: CORRECT — the test uses the production binary, real shell limits, and the command's actual file-output branch without mocks or circular assertions.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Check and propagate the errors from `outFile.Write`, `outFile.WriteString`, and `outFile.Close`, and reject short writes by verifying the returned byte counts match the expected payload length.
