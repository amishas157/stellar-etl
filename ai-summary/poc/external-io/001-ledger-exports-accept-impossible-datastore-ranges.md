# H001: Datastore-backed ledger exports silently accept impossible bounded ranges

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_ledgers` and `export_token_transfer` should reject impossible bounded ranges such as `--end-ledger 0` or `--start-ledger 100 --end-ledger 50` before creating output, because ledger 0 does not exist and a reversed range cannot describe any real Stellar history segment.

## Mechanism

The default path for both commands calls `input.GetLedgers()`, which builds a buffered datastore backend but never runs `ValidateLedgerRange()`. If the backend accepts `ledgerbackend.BoundedRange(start, end)`, the subsequent `for seq := start; seq <= end; seq++` loop simply never executes for `end == 0` or `end < start`, so the command writes an empty success-shaped file and can even upload it to GCS.

## Trigger

1. Run `stellar-etl export_ledgers --start-ledger 100 --end-ledger 50 -o out.json` or `stellar-etl export_token_transfer --start-ledger 2 --end-ledger 0 -o out.json`.
2. Observe that the command reports zero written bytes instead of rejecting the impossible range.

## Target Code

- `internal/input/ledgers.go:GetLedgers:14-90` — prepares a bounded datastore range without validation, then skips the loop when `seq <= end` is false at entry
- `cmd/export_ledgers.go:Run:17-73` — treats an empty `[]HistoryArchiveLedgerAndLCM` as a successful export
- `cmd/export_token_transfers.go:Run:17-63` — shares the same `GetLedgers()` helper and therefore the same empty-success path
- `cmd/export_ledgers_test.go:TestExportLedger:29-48` — currently codifies `end before start` and `end is 0` as zero-byte success cases

## Evidence

`ValidateLedgerRange()` exists in `internal/utils/main.go:735-758`, but `GetLedgers()` does not call it. The only in-repo CLI test coverage for these bad ranges explicitly expects `"Number of bytes written: 0"` rather than an error, showing the current behavior is a silent empty export rather than a rejected request.

## Anti-Evidence

The `--captive-core` history-archive path for ledgers goes through `utils.CreateBackend()`, which does validate bounded ranges first. This may limit the bug to the default datastore-backed path rather than every possible `export_ledgers` configuration.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`GetLedgers()` (internal/input/ledgers.go:14) creates a datastore backend via `CreateLedgerBackend()` (internal/utils/main.go:1011), which builds a `BufferedStorageBackend` without any range validation. The loop `for seq := start; seq <= end; seq++` at line 24 silently produces zero iterations when `end < start` or `end == 0` (with default start=2). The empty slice propagates back to the export command, which writes an empty output file and calls `MaybeUpload()`, potentially pushing empty data to GCS. In contrast, the captive-core path calls `CreateBackend()` (line 760), which calls `ValidateLedgerRange()` (line 770) to reject these cases.

### Code Paths Examined

- `internal/input/ledgers.go:GetLedgers:14-91` — no call to `ValidateLedgerRange()` before loop; empty loop returns empty slice with nil error
- `internal/utils/main.go:CreateLedgerBackend:1011-1041` — builds BufferedStorageBackend, no range validation
- `internal/utils/main.go:CreateBackend:760-770` — captive-core path calls `ValidateLedgerRange()` at line 770 (confirming the validation was intended)
- `internal/input/changes.go:68` — `StreamChanges` also calls `ValidateLedgerRange()`, confirming the pattern is expected
- `internal/utils/main.go:ValidateLedgerRange:736-758` — validates start!=0, end!=0, end>=start, range within latest ledger
- `internal/utils/main.go:AddArchiveFlags:251` — `start-ledger` defaults to 2
- `internal/utils/main.go:AddCommonFlags:233` — `end-ledger` defaults to 0
- `cmd/export_ledgers.go:28-34` — branches on `UseCaptiveCore`; default path calls `GetLedgers()` without validation
- `cmd/export_ledgers.go:63-68` — closes file, logs "Number of bytes written: 0", calls `MaybeUpload()` even on empty export
- `cmd/export_token_transfers.go:28` — always calls `GetLedgers()` (no captive-core branch), same missing validation
- `cmd/export_ledgers_test.go:31-48` — tests "end before start" and "end is 0" expect `"Number of bytes written: 0"`, codifying the silent empty export

### Findings

The bug is confirmed. Two of three callers of range validation (`CreateBackend`, `StreamChanges`) properly call `ValidateLedgerRange()`, but `GetLedgers()` — used by the default code paths for `export_ledgers` and `export_token_transfer` — does not. This is an inconsistency between the captive-core and datastore paths.

Key observations:
1. **`export_token_transfers` is worse**: it always calls `GetLedgers()` with no captive-core fallback, so there is no configuration that enables validation for this command.
2. **Empty files upload to GCS**: both commands call `MaybeUpload()` after the export loop, so an empty file can be pushed to cloud storage, creating silent data gaps in downstream pipelines.
3. **Tests encode the bug**: `export_ledgers_test.go` explicitly asserts "Number of bytes written: 0" for impossible ranges rather than asserting an error, meaning the bug is codified as expected behavior.

### PoC Guidance

- **Test file**: `cmd/export_ledgers_test.go` (append new test cases)
- **Setup**: The existing test infrastructure builds the binary and runs CLI commands. No special setup needed beyond what `TestMain` provides.
- **Steps**:
  1. Add a test case for `export_ledgers` with `--start-ledger 100 --end-ledger 50` that asserts the command exits with a non-zero status or error message containing "end sequence number is less than start"
  2. Add a test case for `export_ledgers` with `--end-ledger 0` that asserts an error message containing "end sequence number equal to 0"
  3. Add a test for `export_token_transfer` with the same impossible ranges
- **Assertion**: The current behavior produces `"Number of bytes written: 0"` with a success exit. The fix would add a `ValidateLedgerRange()` call in `GetLedgers()` (requiring a `latestNum` parameter or a simpler local check for start/end ordering), causing these cases to return an error. Alternatively, add validation at the command level before calling `GetLedgers()`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4-6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestLedgerExportsAcceptImpossibleDatastoreRanges"
**Test Language**: Go

### Demonstration

The test confirms that `ValidateLedgerRange` correctly rejects impossible ranges (end < start, end == 0) with descriptive error messages, but both `export_ledgers` and `export_token_transfer` bypass this validation entirely. When invoked with `--start-ledger 100 --end-ledger 50` or `--end-ledger 0`, both commands exit successfully with "Number of bytes written: 0" instead of returning an error, silently producing empty exports that could be uploaded to GCS as data gaps.

### Test Body

```go
package cmd

import (
	"os/exec"
	"strings"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/utils"
)

// TestLedgerExportsAcceptImpossibleDatastoreRanges demonstrates that export_ledgers
// and export_token_transfer silently accept impossible bounded ranges (end < start,
// end == 0) and produce empty success-shaped output instead of returning an error.
//
// ValidateLedgerRange exists and correctly rejects these ranges, but GetLedgers()
// never calls it — unlike the captive-core path (CreateBackend) and StreamChanges,
// which both validate ranges before processing.
func TestLedgerExportsAcceptImpossibleDatastoreRanges(t *testing.T) {
	// Part 1: Demonstrate that ValidateLedgerRange correctly rejects impossible ranges.
	// This validation function exists but is NOT called in the GetLedgers() code path.
	t.Run("ValidateLedgerRange_rejects_end_before_start", func(t *testing.T) {
		err := utils.ValidateLedgerRange(100, 50, 1000)
		if err == nil {
			t.Fatal("Expected ValidateLedgerRange to reject end < start, but got nil error")
		}
		if !strings.Contains(err.Error(), "end sequence number is less than start") {
			t.Errorf("Unexpected error message: %s", err.Error())
		}
	})

	t.Run("ValidateLedgerRange_rejects_end_is_zero", func(t *testing.T) {
		err := utils.ValidateLedgerRange(2, 0, 1000)
		if err == nil {
			t.Fatal("Expected ValidateLedgerRange to reject end == 0, but got nil error")
		}
		if !strings.Contains(err.Error(), "end sequence number equal to 0") {
			t.Errorf("Unexpected error message: %s", err.Error())
		}
	})

	// Part 2: Demonstrate that the export_ledgers command silently succeeds with
	// impossible ranges instead of returning an error. The binary is built by TestMain.
	t.Run("export_ledgers_silently_succeeds_with_end_before_start", func(t *testing.T) {
		cmd := exec.Command("./stellar-etl", "export_ledgers", "-s", "100", "-e", "50")
		output, err := cmd.CombinedOutput()
		outputStr := string(output)

		// BUG: The command should exit with an error about impossible range,
		// but instead it exits successfully with "Number of bytes written: 0".
		if err != nil {
			t.Fatalf("Command failed unexpectedly (this would mean the bug is fixed): %v\nOutput: %s", err, outputStr)
		}

		if !strings.Contains(outputStr, "Number of bytes written: 0") {
			t.Fatalf("Expected silent empty export, got: %s", outputStr)
		}

		// Verify no error about the invalid range was reported
		if strings.Contains(outputStr, "end sequence number is less than start") {
			t.Fatal("Command correctly rejected impossible range — bug may be fixed")
		}

		t.Logf("BUG CONFIRMED: export_ledgers with --start-ledger 100 --end-ledger 50 "+
			"silently wrote 0 bytes instead of returning an error.\nOutput: %s", outputStr)
	})

	t.Run("export_ledgers_silently_succeeds_with_end_is_zero", func(t *testing.T) {
		cmd := exec.Command("./stellar-etl", "export_ledgers", "-e", "0")
		output, err := cmd.CombinedOutput()
		outputStr := string(output)

		// BUG: The command should exit with an error about end == 0,
		// but instead it exits successfully with "Number of bytes written: 0".
		if err != nil {
			t.Fatalf("Command failed unexpectedly (this would mean the bug is fixed): %v\nOutput: %s", err, outputStr)
		}

		if !strings.Contains(outputStr, "Number of bytes written: 0") {
			t.Fatalf("Expected silent empty export, got: %s", outputStr)
		}

		if strings.Contains(outputStr, "end sequence number equal to 0") {
			t.Fatal("Command correctly rejected end == 0 — bug may be fixed")
		}

		t.Logf("BUG CONFIRMED: export_ledgers with --end-ledger 0 "+
			"silently wrote 0 bytes instead of returning an error.\nOutput: %s", outputStr)
	})

	t.Run("export_token_transfer_silently_succeeds_with_end_before_start", func(t *testing.T) {
		cmd := exec.Command("./stellar-etl", "export_token_transfer", "-s", "100", "-e", "50")
		output, err := cmd.CombinedOutput()
		outputStr := string(output)

		// BUG: The command should exit with an error, but silently succeeds.
		if err != nil {
			t.Fatalf("Command failed unexpectedly (this would mean the bug is fixed): %v\nOutput: %s", err, outputStr)
		}

		if !strings.Contains(outputStr, "Number of bytes written: 0") {
			t.Fatalf("Expected silent empty export, got: %s", outputStr)
		}

		if strings.Contains(outputStr, "end sequence number is less than start") {
			t.Fatal("Command correctly rejected impossible range — bug may be fixed")
		}

		t.Logf("BUG CONFIRMED: export_token_transfer with --start-ledger 100 --end-ledger 50 "+
			"silently wrote 0 bytes instead of returning an error.\nOutput: %s", outputStr)
	})
}
```

### Test Output

```
=== RUN   TestLedgerExportsAcceptImpossibleDatastoreRanges
=== RUN   TestLedgerExportsAcceptImpossibleDatastoreRanges/ValidateLedgerRange_rejects_end_before_start
=== RUN   TestLedgerExportsAcceptImpossibleDatastoreRanges/ValidateLedgerRange_rejects_end_is_zero
=== RUN   TestLedgerExportsAcceptImpossibleDatastoreRanges/export_ledgers_silently_succeeds_with_end_before_start
    data_integrity_poc_test.go:63: BUG CONFIRMED: export_ledgers with --start-ledger 100 --end-ledger 50 silently wrote 0 bytes instead of returning an error.
        Output: time="2026-04-12T02:52:51.454-05:00" level=info msg="Number of bytes written: 0" pid=42547
        time="2026-04-12T02:52:51.455-05:00" level=info msg="{\"attempted_transforms\":0,\"failed_transforms\":0,\"successful_transforms\":0}" pid=42547
        time="2026-04-12T02:52:51.455-05:00" level=info msg="No cloud provider specified for upload. Skipping upload." pid=42547
=== RUN   TestLedgerExportsAcceptImpossibleDatastoreRanges/export_ledgers_silently_succeeds_with_end_is_zero
    data_integrity_poc_test.go:86: BUG CONFIRMED: export_ledgers with --end-ledger 0 silently wrote 0 bytes instead of returning an error.
        Output: time="2026-04-12T02:52:52.314-05:00" level=info msg="Number of bytes written: 0" pid=42548
        time="2026-04-12T02:52:52.314-05:00" level=info msg="{\"attempted_transforms\":0,\"failed_transforms\":0,\"successful_transforms\":0}" pid=42548
        time="2026-04-12T02:52:52.314-05:00" level=info msg="No cloud provider specified for upload. Skipping upload." pid=42548
=== RUN   TestLedgerExportsAcceptImpossibleDatastoreRanges/export_token_transfer_silently_succeeds_with_end_before_start
    data_integrity_poc_test.go:108: BUG CONFIRMED: export_token_transfer with --start-ledger 100 --end-ledger 50 silently wrote 0 bytes instead of returning an error.
        Output: time="2026-04-12T02:52:53.107-05:00" level=info msg="Number of bytes written: 0" pid=42549
        time="2026-04-12T02:52:53.107-05:00" level=info msg="{\"attempted_transforms\":0,\"failed_transforms\":0,\"successful_transforms\":0}" pid=42549
        time="2026-04-12T02:52:53.107-05:00" level=info msg="No cloud provider specified for upload. Skipping upload." pid=42549
--- PASS: TestLedgerExportsAcceptImpossibleDatastoreRanges (3.26s)
    --- PASS: TestLedgerExportsAcceptImpossibleDatastoreRanges/ValidateLedgerRange_rejects_end_before_start (0.00s)
    --- PASS: TestLedgerExportsAcceptImpossibleDatastoreRanges/ValidateLedgerRange_rejects_end_is_zero (0.00s)
    --- PASS: TestLedgerExportsAcceptImpossibleDatastoreRanges/export_ledgers_silently_succeeds_with_end_before_start (1.61s)
    --- PASS: TestLedgerExportsAcceptImpossibleDatastoreRanges/export_ledgers_silently_succeeds_with_end_is_zero (0.86s)
    --- PASS: TestLedgerExportsAcceptImpossibleDatastoreRanges/export_token_transfer_silently_succeeds_with_end_before_start (0.79s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	8.935s
```
