# 015: `export_ledgers` and `export_token_transfer` accept impossible datastore ranges

**Date**: 2026-04-12
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

The default datastore-backed paths for `export_ledgers` and `export_token_transfer` accept impossible bounded ledger ranges such as `--end-ledger 0` and `--start-ledger 100 --end-ledger 50`. Instead of rejecting those requests, both commands emit an empty output file, log `Number of bytes written: 0`, and continue through the normal success path.

This is inconsistent with the shared `ValidateLedgerRange()` helper and the captive-core / change-stream paths, which explicitly reject the same impossible ranges. The result is a silent empty export that looks successful and can propagate downstream as a real data gap.

## Root Cause

`input.GetLedgers()` creates a datastore-backed ledger backend and immediately calls `PrepareRange(BoundedRange(start, end))`, but it never validates `start` and `end` first. When `end < start` or `end == 0`, the subsequent `for seq := start; seq <= end; seq++` loop executes zero times, so the helper returns an empty slice and nil error.

Both `cmd/export_ledgers.go` and `cmd/export_token_transfers.go` treat that empty slice as a successful export: they create the output file, write zero rows, log success-shaped stats, and call `MaybeUpload()`.

## Reproduction

During normal operation, a caller can request an impossible bounded range on the default datastore path, for example:

- `stellar-etl export_ledgers --start-ledger 100 --end-ledger 50 -o out.json`
- `stellar-etl export_ledgers --end-ledger 0 -o out.json`
- `stellar-etl export_token_transfer --start-ledger 100 --end-ledger 50 -o out.json`
- `stellar-etl export_token_transfer --end-ledger 0 -o out.json`

Each command exits successfully, creates an empty output file, and logs `Number of bytes written: 0` instead of rejecting the impossible request.

## Affected Code

- `internal/input/ledgers.go:GetLedgers:14-90` — prepares a bounded datastore range without validating impossible inputs, then returns an empty slice when the loop never runs
- `cmd/export_ledgers.go:17-73` — treats the empty result as a successful export and proceeds to optional upload
- `cmd/export_token_transfers.go:17-63` — shares the same empty-success behavior through `GetLedgers()`
- `internal/utils/main.go:736-758` — shared validator that would reject `end == 0` and `end < start`
- `internal/utils/main.go:760-770` — captive-core/history-archive path showing the intended validation pattern

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestLedgerExportsAcceptImpossibleDatastoreRanges`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create test file at `cmd/data_integrity_poc_test.go`
  3. Run: `go test ./cmd/... -run '^TestLedgerExportsAcceptImpossibleDatastoreRanges$' -v`
  4. Observe: each command exits successfully, logs `Number of bytes written: 0`, and leaves an empty output file instead of rejecting the invalid range

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

func runLedgerExportPoC(t *testing.T, args ...string) (string, error) {
	t.Helper()

	cmd := exec.Command("./stellar-etl", args...)
	output, err := cmd.CombinedOutput()
	return string(output), err
}

// TestLedgerExportsAcceptImpossibleDatastoreRanges demonstrates that the
// datastore-backed ledger export commands silently accept impossible bounded
// ranges and emit empty success-shaped output instead of rejecting them.
func TestLedgerExportsAcceptImpossibleDatastoreRanges(t *testing.T) {
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
			name:          "export_ledgers accepts end before start",
			args:          []string{"export_ledgers", "-s", "100", "-e", "50"},
			wantValidator: "end sequence number is less than start",
		},
		{
			name:          "export_ledgers accepts end is zero",
			args:          []string{"export_ledgers", "-e", "0"},
			wantValidator: "end sequence number equal to 0",
		},
		{
			name:          "export_token_transfer accepts end before start",
			args:          []string{"export_token_transfer", "-s", "100", "-e", "50"},
			wantValidator: "end sequence number is less than start",
		},
		{
			name:          "export_token_transfer accepts end is zero",
			args:          []string{"export_token_transfer", "-e", "0"},
			wantValidator: "end sequence number equal to 0",
		},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			outputPath := filepath.Join(t.TempDir(), "out.txt")
			args := append(tc.args, "-o", outputPath)

			output, err := runLedgerExportPoC(t, args...)
			if err != nil {
				t.Fatalf("command failed unexpectedly; datastore path may now reject this invalid range: %v\nOutput: %s", err, output)
			}

			if !strings.Contains(output, "Number of bytes written: 0") {
				t.Fatalf("expected silent empty export, got: %s", output)
			}

			if strings.Contains(output, tc.wantValidator) {
				t.Fatalf("command reported validation error instead of silently succeeding: %s", output)
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

- **Expected**: Datastore-backed bounded ledger exports should reject impossible ranges like `end == 0` and `end < start`, matching the shared validator and other validated reader paths.
- **Actual**: `export_ledgers` and `export_token_transfer` accept those ranges, emit empty files, log zero written bytes, and continue as though the export succeeded.

## Adversarial Review

1. Exercises claimed bug: YES — the final PoC runs the production CLI commands and proves they succeed on invalid bounded ranges while leaving empty output files behind.
2. Realistic preconditions: YES — the trigger uses only documented CLI flags; no internal APIs, mocks, or impossible ledger data are required.
3. Bug vs by-design: BUG — `ValidateLedgerRange()` already rejects these ranges, and sibling paths (`CreateBackend`, `StreamChanges`) explicitly apply that validation before bounded reads.
4. Final severity: Medium — the exporter returns silent empty output for a request that should fail, which can create downstream data gaps without any explicit error.
5. In scope: YES — this is a concrete exporter correctness failure that produces plausible-but-wrong output artifacts rather than crashing.
6. Test correctness: CORRECT — the PoC checks both the intended validator behavior and the production command behavior, uses temporary output paths to avoid stale-file artifacts, and confirms the output file is actually empty.
7. Alternative explanations: NONE — the empty success path is explained directly by missing validation plus the zero-iteration loop in `GetLedgers()`.
8. Novelty: NOVEL

## Suggested Fix

Reject impossible bounded ranges before datastore-backed ledger reads. The minimal safe fix is to validate `start != 0`, `end != 0`, and `end >= start` in `GetLedgers()` or at the two command call sites before creating the output file. If the project also wants parity with captive-core validation for future-ledger bounds, fetch the latest ledger sequence and apply the full `ValidateLedgerRange()` helper there as well.
