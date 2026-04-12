# 018: `export_assets` accepts impossible datastore ranges

**Date**: 2026-04-12
**Severity**: High
**Impact**: structural data corruption
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

The default datastore-backed `export_assets` command accepts impossible bounded ledger ranges such as `--start-ledger 100 --end-ledger 50` and `--start-ledger 100 --end-ledger 0`. Instead of rejecting those requests, it creates an empty output file, logs success-shaped zero-row stats, and exits with status `0`.

The root cause is a missing validation step on the datastore reader path. `GetPaymentOperations()` prepares a bounded backend range and then iterates `for seq := start; seq <= end; seq++`; when `end < start` or `end == 0`, that loop never runs, so the helper returns an empty slice and nil error, which `export_assets` treats as a legitimate empty export.

## Root Cause

`input.GetPaymentOperations()` uses `CreateLedgerBackend()` instead of the validating history-archive helper path. `CreateLedgerBackend()` never calls `ValidateLedgerRange()`, so impossible bounded ranges survive until the reader's outer ledger loop. Because the loop condition is false immediately, the function returns `[]AssetTransformInput{}` with no error.

`cmd/export_assets.go` then opens the requested output path, skips its transform/write loop because the input slice is empty, closes the file, logs `0 bytes written`, emits zero-row stats, and can optionally upload the empty artifact.

## Reproduction

During normal CLI usage on the default datastore-backed path, run either of the following:

- `stellar-etl export_assets --start-ledger 100 --end-ledger 50 -o out.json`
- `stellar-etl export_assets --start-ledger 100 --end-ledger 0 -o out.json`

Each command exits successfully and leaves an empty output file instead of rejecting the invalid range.

## Affected Code

- `cmd/export_assets.go:Run:25-37` — opens the output file and dispatches to the datastore-backed `GetPaymentOperations()` path by default
- `cmd/export_assets.go:Run:44-77` — treats an empty `paymentOps` slice as a successful export, logs zero bytes written, and proceeds to optional upload
- `internal/input/assets.go:GetPaymentOperations:21-60` — prepares a bounded range without validation and returns an empty slice when `for seq := start; seq <= end; seq++` never executes
- `internal/utils/main.go:ValidateLedgerRange:736-758` — existing shared validator that would reject `end == 0` and `end < start`
- `internal/utils/main.go:CreateLedgerBackend:1011-1040` — datastore backend factory used by `GetPaymentOperations()`, with no bounded-range validation
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:13-16` — contrasting history-archive path that calls `CreateBackend()`, which does validate bounded ranges

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestExportAssetsAcceptsImpossibleDatastoreRanges`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create test file at `cmd/data_integrity_poc_test.go`
  3. Run: `go test ./cmd/... -run '^TestExportAssetsAcceptsImpossibleDatastoreRanges$' -v`
  4. Observe: both invalid ranges exit successfully and leave empty output files instead of being rejected

### Test Body

```go
package cmd

import (
	"os"
	"os/exec"
	"path/filepath"
	"strings"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/utils"
)

func runAssetExportPoC(t *testing.T, args ...string) (string, error) {
	t.Helper()

	cmd := exec.Command("./stellar-etl", args...)
	output, err := cmd.CombinedOutput()
	return string(output), err
}

func TestExportAssetsAcceptsImpossibleDatastoreRanges(t *testing.T) {
	t.Run("ValidateLedgerRange rejects impossible ranges", func(t *testing.T) {
		t.Run("end before start", func(t *testing.T) {
			err := utils.ValidateLedgerRange(100, 50, 1000)
			if err == nil {
				t.Fatal("expected ValidateLedgerRange to reject end < start")
			}
			if !strings.Contains(err.Error(), "end sequence number is less than start") {
				t.Fatalf("unexpected validation error: %v", err)
			}
		})

		t.Run("end is zero", func(t *testing.T) {
			err := utils.ValidateLedgerRange(2, 0, 1000)
			if err == nil {
				t.Fatal("expected ValidateLedgerRange to reject end == 0")
			}
			if !strings.Contains(err.Error(), "end sequence number equal to 0") {
				t.Fatalf("unexpected validation error: %v", err)
			}
		})
	})

	cases := []struct {
		name          string
		args          []string
		wantValidator string
	}{
		{
			name:          "export_assets accepts end before start",
			args:          []string{"export_assets", "-s", "100", "-e", "50"},
			wantValidator: "end sequence number is less than start",
		},
		{
			name:          "export_assets accepts end is zero",
			args:          []string{"export_assets", "-s", "100", "-e", "0"},
			wantValidator: "end sequence number equal to 0",
		},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			outputPath := filepath.Join(t.TempDir(), "out.json")
			args := append(tc.args, "-o", outputPath)

			output, err := runAssetExportPoC(t, args...)
			if err != nil {
				t.Fatalf("command failed unexpectedly; invalid ranges may now be rejected: %v\nOutput: %s", err, output)
			}

			if strings.Contains(output, tc.wantValidator) {
				t.Fatalf("command reported validation error instead of silently succeeding: %s", output)
			}
			if !strings.Contains(output, "0 bytes written") {
				t.Fatalf("expected success-shaped empty export log, got: %s", output)
			}

			info, statErr := os.Stat(outputPath)
			if statErr != nil {
				t.Fatalf("expected output file to be created: %v", statErr)
			}
			if info.Size() != 0 {
				t.Fatalf("expected empty output file, got %d bytes", info.Size())
			}
		})
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `export_assets` should reject impossible bounded ranges like `end == 0` and `end < start`, consistent with `ValidateLedgerRange()` and the history-archive path.
- **Actual**: the default datastore-backed command accepts those ranges, writes an empty file, logs success-shaped zero-row output, and exits `0`.

## Adversarial Review

1. Exercises claimed bug: YES — the final PoC runs the production CLI command and proves both impossible bounded ranges exit successfully while leaving empty output files behind.
2. Realistic preconditions: YES — the trigger uses only documented CLI flags and the default datastore-backed execution path; no mocks, test-only APIs, or fabricated ledgers are required.
3. Bug vs by-design: BUG — the codebase already includes `ValidateLedgerRange()` and uses it on the history-archive asset path, so silently treating invalid bounded ranges as empty exports is inconsistent with existing contract enforcement.
4. Final severity: High — this is a success-shaped structural corruption bug: an invalid request produces a plausible empty artifact that downstream jobs can misread as “no assets in range.”
5. In scope: YES — this is a concrete data-correctness failure in a production export command.
6. Test correctness: CORRECT — the PoC uses temporary output paths, checks both validator behavior and production command behavior, and confirms the created files are actually empty.
7. Alternative explanations: NONE — the observed behavior follows directly from missing validation plus the zero-iteration ledger loop in `GetPaymentOperations()`.
8. Novelty: NOVEL

## Suggested Fix

Reject impossible bounded ranges before calling `PrepareRange(BoundedRange(start, end))` in `GetPaymentOperations()`. The safest repair is to align the datastore-backed asset reader with the validated history-archive path by applying `ValidateLedgerRange()` before the export opens output files; at minimum, reject `start == 0`, `end == 0`, and `end < start`.
