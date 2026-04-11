# 011: `export_token_transfer` silently ignores requested Parquet output

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Subsystem**: cli-commands
**Final review by**: gpt-5.4, high

## Summary

`export_token_transfer` accepts `--write-parquet` and `--parquet-output`, but a real command run exits successfully after writing only the JSON export. The requested parquet artifact is never created, so a caller can believe a dual-format export succeeded when the parquet side was silently dropped.

This was confirmed with an end-to-end reproduction against a real ledger range, not just source inspection. JSON output was produced, exit status was 0, and the requested parquet path remained absent.

## Root Cause

The command parses the shared parquet flags but discards the returned `parquetPath` and never checks `commonArgs.WriteParquet`. It always follows the JSON-only export path and then optionally uploads only the JSON file. The transform layer also has no token-transfer parquet schema/converter wired in, so there is no implementation path that could satisfy the advertised parquet request.

## Reproduction

Run `export_token_transfer --write-parquet --parquet-output <path>` over any non-empty ledger range. During normal execution the command fetches ledgers, transforms token-transfer events, writes JSON rows to `--output`, closes that file, logs success-style stats, and exits 0 without ever branching into parquet generation. The requested parquet file is left missing even though the CLI accepted the parquet flags.

## Affected Code

- `internal/utils/main.go:232-245` — shared archive commands register `--write-parquet`
- `internal/utils/main.go:541-563` — shared archive commands parse and return `parquet-output`
- `cmd/export_token_transfers.go:17-21` — parses flags but discards `parquetPath`
- `cmd/export_token_transfers.go:34-63` — writes JSON output and uploads it, but never checks `commonArgs.WriteParquet` or calls `WriteParquet`
- `internal/transform/schema.go:659-677` — token-transfer output exists only as the JSON struct today

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestTokenTransferWriteParquetNoop`
- **Test language**: go
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package cmd

import (
	"os"
	"path/filepath"
	"testing"

	"github.com/stretchr/testify/require"
)

func TestTokenTransferWriteParquetNoop(t *testing.T) {
	jsonPath := filepath.Join(t.TempDir(), "token-transfers.json")
	parquetPath := filepath.Join(t.TempDir(), "token-transfers.parquet")

	rootCmd.SetArgs([]string{
		"export_token_transfer",
		"-s", "30820015",
		"-e", "30820015",
		"-o", jsonPath,
		"--write-parquet",
		"--parquet-output", parquetPath,
	})

	err := rootCmd.Execute()
	require.NoError(t, err)

	jsonInfo, err := os.Stat(jsonPath)
	require.NoError(t, err)
	require.Greater(t, jsonInfo.Size(), int64(0))

	_, err = os.Stat(parquetPath)
	require.Error(t, err)
	require.True(t, os.IsNotExist(err), "expected parquet output to be missing despite --write-parquet")
}
```

## Expected vs Actual Behavior

- **Expected**: when `--write-parquet` is requested, `export_token_transfer` should either emit a parquet file at `--parquet-output` or fail immediately with a clear unsupported-format error.
- **Actual**: the command exits successfully, writes only JSON, and leaves the requested parquet path missing.

## Adversarial Review

1. Exercises claimed bug: YES — the final PoC runs the production command path against a real ledger range and checks the actual filesystem outputs.
2. Realistic preconditions: YES — the parquet flags are public CLI options on this command, and the tested ledger range is the same class of normal export input already covered by existing token-transfer tests.
3. Bug vs by-design: BUG — the command advertises and accepts parquet flags through shared helpers and gives no warning or error that parquet is unsupported.
4. Final severity: Medium — this does not corrupt row contents, but it silently drops a requested export artifact while reporting success.
5. In scope: YES — this is a concrete export-path correctness bug that can leave downstream parquet consumers with missing data.
6. Test correctness: CORRECT — the original PoC only inspected source text, so the final review rewrote it into an end-to-end runtime check that proves the missing artifact under real execution.
7. Alternative explanations: NONE — the requested parquet path is ignored because the command never uses it.
8. Novelty: NOVEL

## Suggested Fix

Either implement token-transfer parquet support end-to-end (schema, converter, command wiring) or reject `--write-parquet`/`--parquet-output` for this command with a fatal unsupported-format error before any export begins. Keep a regression test that proves the command cannot exit 0 while dropping the requested parquet file.
