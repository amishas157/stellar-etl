# 010: Shared parquet flags are silently ignored by `export_token_transfer` and `export_ledger_transaction`

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`export_token_transfer` and `export_ledger_transaction` both inherit the shared `--write-parquet` and `--parquet-output` flags, accept them without error, exit successfully, and still skip the requested Parquet artifact entirely. A corrected runtime PoC shows both commands writing non-empty JSON while leaving the requested `.parquet` path absent, whereas sibling command `export_assets` creates the Parquet file under the same conditions.

The original PoC was not sufficient because it only inspected source text and targeted a non-existent test file. After replacing it with a build-and-run command-level test, the silent-ignore behavior is directly reproducible on production code paths.

## Root Cause

Both commands call `utils.AddCommonFlags()` and `utils.AddArchiveFlags()`, so Cobra exposes `--write-parquet` and `--parquet-output` to users. But their `Run` functions discard `parquetPath` from `MustArchiveFlags()` with `_`, never branch on `commonArgs.WriteParquet`, and never call `WriteParquet()` or `MaybeUpload()` for a Parquet artifact.

## Reproduction

During normal operation, a caller can invoke either command with `--write-parquet --parquet-output <path>`. The command succeeds, writes its JSON export, logs no warning that Parquet is unsupported, and leaves the requested Parquet file missing, so downstream automation receives only a partial artifact set despite a successful exit status.

## Affected Code

- `internal/utils/main.go:AddCommonFlags:232-246` — registers the shared `--write-parquet` flag for all archive exporters
- `internal/utils/main.go:AddArchiveFlags:248-255` — registers the shared `--parquet-output` flag and default path
- `internal/utils/main.go:MustCommonFlags:460-537` — parses `write-parquet` into `CommonFlagValues`
- `internal/utils/main.go:MustArchiveFlags:540-563` — returns `parquetPath` to callers
- `cmd/export_token_transfers.go:Run:17-63` — discards `parquetPath`, writes JSON only, and never checks `commonArgs.WriteParquet`
- `cmd/export_ledger_transaction.go:Run:17-57` — discards `parquetPath`, writes JSON only, and never checks `commonArgs.WriteParquet`

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestParquetFlagsSilentlyIgnoredByTokenTransferAndLedgerTransaction`
- **Test language**: `go`
- **How to run**: Create the target test file with the body below, then run `go test ./cmd/... -run TestParquetFlagsSilentlyIgnoredByTokenTransferAndLedgerTransaction -v`.

### Test Body

```go
package cmd

import (
	"bytes"
	"os"
	"os/exec"
	"path/filepath"
	"runtime"
	"strings"
	"testing"
)

func TestParquetFlagsSilentlyIgnoredByTokenTransferAndLedgerTransaction(t *testing.T) {
	_, thisFile, _, ok := runtime.Caller(0)
	if !ok {
		t.Fatal("could not determine current file path")
	}
	repoRoot := filepath.Dir(filepath.Dir(thisFile))
	workDir := t.TempDir()
	binaryPath := filepath.Join(workDir, "stellar-etl")

	build := exec.Command("go", "build", "-o", binaryPath, repoRoot)
	build.Dir = repoRoot
	buildOutput, err := build.CombinedOutput()
	if err != nil {
		t.Fatalf("failed to build stellar-etl: %v\n%s", err, buildOutput)
	}

	type commandCase struct {
		name           string
		args           []string
		expectParquet  bool
		expectJSONData bool
	}

	cases := []commandCase{
		{
			name: "token transfer ignores parquet",
			args: []string{
				"export_token_transfer",
				"-s", "30820015",
				"-e", "30820015",
			},
			expectParquet:  false,
			expectJSONData: true,
		},
		{
			name: "ledger transaction ignores parquet",
			args: []string{
				"export_ledger_transaction",
				"-s", "30820015",
				"-e", "30820015",
			},
			expectParquet:  false,
			expectJSONData: true,
		},
		{
			name: "assets writes parquet when requested",
			args: []string{
				"export_assets",
				"-s", "30820015",
				"-e", "30820015",
			},
			expectParquet:  true,
			expectJSONData: true,
		},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			jsonPath := filepath.Join(workDir, strings.ReplaceAll(tc.name, " ", "_")+".json")
			parquetPath := filepath.Join(workDir, strings.ReplaceAll(tc.name, " ", "_")+".parquet")

			args := append([]string{}, tc.args...)
			args = append(args,
				"-o", jsonPath,
				"--write-parquet",
				"--parquet-output", parquetPath,
			)

			cmd := exec.Command(binaryPath, args...)
			var stdout bytes.Buffer
			var stderr bytes.Buffer
			cmd.Stdout = &stdout
			cmd.Stderr = &stderr

			if err := cmd.Run(); err != nil {
				t.Fatalf("command failed: %v\nstdout:\n%s\nstderr:\n%s", err, stdout.String(), stderr.String())
			}

			jsonInfo, err := os.Stat(jsonPath)
			if err != nil {
				t.Fatalf("expected JSON output file %q to exist: %v\nstderr:\n%s", jsonPath, err, stderr.String())
			}
			if tc.expectJSONData && jsonInfo.Size() == 0 {
				t.Fatalf("expected JSON output file %q to be non-empty", jsonPath)
			}

			parquetInfo, err := os.Stat(parquetPath)
			if tc.expectParquet {
				if err != nil {
					t.Fatalf("expected parquet output file %q to exist: %v\nstderr:\n%s", parquetPath, err, stderr.String())
				}
				if parquetInfo.Size() == 0 {
					t.Fatalf("expected parquet output file %q to be non-empty", parquetPath)
				}
				return
			}

			if !os.IsNotExist(err) {
				t.Fatalf("expected parquet output file %q to be absent, got err=%v size=%d", parquetPath, err, parquetInfo.Size())
			}
			if strings.Contains(stderr.String(), "parquet") {
				t.Fatalf("expected silent ignore of parquet flags, but stderr mentioned parquet:\n%s", stderr.String())
			}
		})
	}
}
```

## Expected vs Actual Behavior

- **Expected**: When `--write-parquet --parquet-output <path>` is supplied, the command should either generate the requested Parquet artifact or fail loudly that Parquet is unsupported.
- **Actual**: Both commands accept the flags, exit successfully, write JSON, and silently leave the requested Parquet output path absent.

## Adversarial Review

1. Exercises claimed bug: YES — the final PoC builds the binary and runs the real Cobra commands, proving the requested Parquet artifact is skipped on successful command completion.
2. Realistic preconditions: YES — this uses only documented CLI flags and the same ledger fixtures already used by neighboring command tests.
3. Bug vs by-design: BUG — the commands publicly expose the shared Parquet flags and parse them, so silently discarding the requested artifact violates the CLI contract more than an internal unimplemented feature toggle.
4. Final severity: Medium — row contents in JSON remain correct, but the exporter silently produces an incomplete output set under a successful exit status.
5. In scope: YES — this is a concrete external-output correctness bug that can mislead downstream automation.
6. Test correctness: CORRECT — the PoC checks successful execution, non-empty JSON, absent Parquet for the two target commands, and a positive Parquet control case via `export_assets`.
7. Alternative explanations: NONE — the same environment and invocation create Parquet successfully for `export_assets`, so the missing files are specific to the two target commands.
8. Novelty: NOVEL

## Suggested Fix

Either wire full Parquet support into both commands by adding `SchemaParquet` conversions and `WriteParquet()` plumbing, or stop registering `--write-parquet` and `--parquet-output` for commands that do not support that output. In either case, a successful command should never silently ignore a user-requested export artifact.
