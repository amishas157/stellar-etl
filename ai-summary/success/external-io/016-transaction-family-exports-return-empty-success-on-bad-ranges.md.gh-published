# 016: Transaction-family exports return empty success on bad ranges

**Date**: 2026-04-12
**Severity**: High
**Impact**: structural data corruption
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`export_transactions`, `export_operations`, `export_effects`, `export_contract_events`, `export_trades`, and `export_ledger_transaction` all accept impossible bounded ledger ranges such as `--start-ledger 100 --end-ledger 50` and `--start-ledger 2 --end-ledger 0`. Instead of rejecting those requests, they create empty output files and exit through the normal success path.

The root cause is shared: the transaction-family readers prepare a bounded backend range but never apply the existing `ValidateLedgerRange()` helper before entering a `for seq := start; seq <= end; seq++` loop. When the loop condition is false from the start, the readers return empty slices and nil errors, so the export commands emit plausible empty artifacts that downstream jobs can mistake for genuine "no rows" results.

## Root Cause

`input.GetTransactions()`, `input.GetOperations()`, and `input.GetTrades()` each call `CreateLedgerBackend()` and `PrepareRange(BoundedRange(start, end))` without validating `start` and `end`. For impossible bounded ranges (`end == 0` or `end < start`), the subsequent outer ledger loop executes zero times, so the helper returns an empty slice and nil error.

All six export commands treat that empty slice as an ordinary successful export: they create the output file, write zero rows, close the file, print success-shaped stats, and optionally upload the empty artifact to cloud storage.

## Reproduction

During normal CLI usage, run any affected command with an impossible bounded range, for example:

- `stellar-etl export_transactions --start-ledger 100 --end-ledger 50 -o out.json`
- `stellar-etl export_operations --start-ledger 2 --end-ledger 0 -o out.json`
- `stellar-etl export_trades --start-ledger 100 --end-ledger 50 -o out.json`

Each command exits successfully and leaves an empty output file instead of rejecting the invalid request.

## Affected Code

- `internal/input/transactions.go:GetTransactions:23-70` — prepares a bounded range without validating impossible inputs, then returns an empty slice when the loop never runs
- `internal/input/operations.go:GetOperations:30-80` — same skipped-loop behavior for operation exports
- `internal/input/trades.go:GetTrades:26-85` — same skipped-loop behavior for trade exports
- `internal/utils/main.go:736-758` — shared validator that would reject `end == 0` and `end < start`
- `internal/utils/main.go:1011-1041` — ledger-backend factory used by the affected readers, with no bounded-range validation
- `cmd/export_transactions.go:25-66` — treats empty transaction input as success
- `cmd/export_operations.go:25-66` — treats empty operation input as success
- `cmd/export_effects.go:25-68` — derives zero effects from empty transaction input and exits successfully
- `cmd/export_contract_events.go:25-65` — derives zero events from empty transaction input and exits successfully
- `cmd/export_trades.go:28-70` — treats empty trade input as success
- `cmd/export_ledger_transaction.go:25-56` — treats empty transaction input as success

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestTransactionFamilyExportsReturnEmptySuccessOnBadRanges`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create test file at `cmd/data_integrity_poc_test.go`
  3. Run: `go test ./cmd/... -run '^TestTransactionFamilyExportsReturnEmptySuccessOnBadRanges$' -v`
  4. Observe: each command exits successfully and leaves an empty output file instead of rejecting the invalid range

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

func runTransactionFamilyExportPoC(t *testing.T, args ...string) (string, error) {
	t.Helper()

	cmd := exec.Command("./stellar-etl", args...)
	output, err := cmd.CombinedOutput()
	return string(output), err
}

// TestTransactionFamilyExportsReturnEmptySuccessOnBadRanges demonstrates that
// transaction-derived export commands silently accept impossible bounded ranges
// and emit empty success-shaped output instead of rejecting them.
func TestTransactionFamilyExportsReturnEmptySuccessOnBadRanges(t *testing.T) {
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
			name:          "export_transactions accepts end before start",
			args:          []string{"export_transactions", "-s", "100", "-e", "50"},
			wantValidator: "end sequence number is less than start",
		},
		{
			name:          "export_transactions accepts end is zero",
			args:          []string{"export_transactions", "-s", "2", "-e", "0"},
			wantValidator: "end sequence number equal to 0",
		},
		{
			name:          "export_operations accepts end before start",
			args:          []string{"export_operations", "-s", "100", "-e", "50"},
			wantValidator: "end sequence number is less than start",
		},
		{
			name:          "export_operations accepts end is zero",
			args:          []string{"export_operations", "-s", "2", "-e", "0"},
			wantValidator: "end sequence number equal to 0",
		},
		{
			name:          "export_effects accepts end before start",
			args:          []string{"export_effects", "-s", "100", "-e", "50"},
			wantValidator: "end sequence number is less than start",
		},
		{
			name:          "export_effects accepts end is zero",
			args:          []string{"export_effects", "-s", "2", "-e", "0"},
			wantValidator: "end sequence number equal to 0",
		},
		{
			name:          "export_contract_events accepts end before start",
			args:          []string{"export_contract_events", "-s", "100", "-e", "50"},
			wantValidator: "end sequence number is less than start",
		},
		{
			name:          "export_contract_events accepts end is zero",
			args:          []string{"export_contract_events", "-s", "2", "-e", "0"},
			wantValidator: "end sequence number equal to 0",
		},
		{
			name:          "export_trades accepts end before start",
			args:          []string{"export_trades", "-s", "100", "-e", "50"},
			wantValidator: "end sequence number is less than start",
		},
		{
			name:          "export_trades accepts end is zero",
			args:          []string{"export_trades", "-s", "2", "-e", "0"},
			wantValidator: "end sequence number equal to 0",
		},
		{
			name:          "export_ledger_transaction accepts end before start",
			args:          []string{"export_ledger_transaction", "-s", "100", "-e", "50"},
			wantValidator: "end sequence number is less than start",
		},
		{
			name:          "export_ledger_transaction accepts end is zero",
			args:          []string{"export_ledger_transaction", "-s", "2", "-e", "0"},
			wantValidator: "end sequence number equal to 0",
		},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			outputPath := filepath.Join(t.TempDir(), "out.json")
			args := append(tc.args, "-o", outputPath)

			output, err := runTransactionFamilyExportPoC(t, args...)
			if err != nil {
				t.Fatalf("command failed unexpectedly; invalid ranges may now be rejected: %v\nOutput: %s", err, output)
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

- **Expected**: Bounded transaction-family exports should reject impossible ranges like `end == 0` and `end < start`, matching the existing `ValidateLedgerRange()` helper and sibling validated reader paths.
- **Actual**: All six commands accept those ranges, create empty output files, and exit successfully as if the export legitimately contained no rows.

## Adversarial Review

1. Exercises claimed bug: YES — the final PoC runs the production CLI commands and proves they succeed on impossible bounded ranges while leaving empty output files behind.
2. Realistic preconditions: YES — the trigger uses only documented CLI flags and the default datastore-backed execution path; no internal APIs, mocks, or fabricated ledger contents are required.
3. Bug vs by-design: BUG — the codebase already has `ValidateLedgerRange()` and uses it in sibling bounded-read paths, while the README describes these commands as bounded-range exports rather than "empty on invalid range" no-ops.
4. Final severity: High — the exporter returns structurally wrong output (an empty artifact for an invalid request) through the normal success path, which can be consumed downstream as a genuine data gap.
5. In scope: YES — this is a concrete data-correctness failure in production export commands.
6. Test correctness: CORRECT — the PoC uses temporary output paths to avoid stale-file artifacts, checks both validator behavior and production command behavior, and confirms the generated files are actually empty.
7. Alternative explanations: NONE — the observed behavior follows directly from missing validation plus the zero-iteration ledger loop in the reader helpers.
8. Novelty: NOVEL

## Suggested Fix

Reject impossible bounded ranges before calling `PrepareRange(BoundedRange(start, end))` in `GetTransactions()`, `GetOperations()`, and `GetTrades()`. The safest fix is to fetch the latest ledger and apply the existing `ValidateLedgerRange()` helper so these readers match other validated bounded-export paths; at minimum, reject `start == 0`, `end == 0`, and `end < start` before creating output files.
