# H002: Datastore-backed `export_assets` accepts impossible ledger ranges and emits empty success

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Structural data corruption: impossible asset ranges can be exported as plausible empty files
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a bounded asset export is impossible — for example `end == 0` or `end < start` — the command should fail before producing output. The repository already has a shared `ValidateLedgerRange()` helper for this exact contract, and a bounded export should not silently translate invalid caller input into "no assets existed in this range."

## Mechanism

The default datastore path in `export_assets` goes through `GetPaymentOperations()`, which uses `CreateLedgerBackend()` instead of the validating history-archive helper. That backend path never calls `ValidateLedgerRange()`, and the reader then loops `for seq := start; seq <= end; seq++`; impossible ranges simply skip the loop and return an empty slice with no error. `cmd/export_assets` treats that empty slice as a successful export and can even upload the empty artifact, creating a success-shaped but false "no rows" result.

## Trigger

1. Run the default datastore-backed command without `--captive-core`, for example:
   - `stellar-etl export_assets --start-ledger 100 --end-ledger 50 -o /tmp/assets.json`
   - `stellar-etl export_assets --start-ledger 100 --end-ledger 0 -o /tmp/assets.json`
2. Inspect the command result and output file.
3. The command should fail validation, but the current path can succeed with an empty asset file and normal success stats.

## Target Code

- `cmd/export_assets.go:Run:30-36` — default path chooses `input.GetPaymentOperations(...)` whenever `--captive-core` is not set.
- `internal/input/assets.go:GetPaymentOperations:21-60` — prepares a bounded range and then iterates `for seq := start; seq <= end; seq++` with no explicit range validation.
- `internal/utils/main.go:CreateLedgerBackend:1011-1041` — datastore/captive-core backend creation path does not call `ValidateLedgerRange()`.
- `internal/utils/main.go:ValidateLedgerRange:736-758` — existing shared validator that would reject `end == 0` and `end < start`.
- `internal/utils/main.go:CreateBackend:760-778` — contrasting history-archive path that does validate bounded ranges.

## Evidence

This is the same empty-success control-flow shape already confirmed for the transaction-family and ledger-family readers, but `export_assets` is not covered by those findings. The split between `CreateLedgerBackend()` (no validation) and `CreateBackend()` (calls `ValidateLedgerRange()`) makes the bug backend-specific and reachable in the command's default path today.

## Anti-Evidence

The `--captive-core` branch of `export_assets` uses `GetPaymentOperationsHistoryArchive()`, which reaches `CreateBackend()` and therefore rejects impossible bounded ranges correctly. A reviewer could argue this limits the blast radius, but the default datastore path is the ordinary production configuration, so the silent empty-success behavior is still live for normal users.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated for `export_assets` specifically; success entries 015 and 016 cover 8 other commands but omit this one

### Trace Summary

`cmd/export_assets.go` line 32 calls `input.GetPaymentOperations(startNum, commonArgs.EndNum, ...)` when `useCaptiveCore` is false (the default). `GetPaymentOperations()` at `internal/input/assets.go:21-61` creates a backend via `utils.CreateLedgerBackend()` (line 23), which at `internal/utils/main.go:1011-1041` never calls `ValidateLedgerRange()`. The function then calls `backend.PrepareRange(ctx, ledgerbackend.BoundedRange(start, end))` and enters `for seq := start; seq <= end; seq++`. For impossible ranges (`end < start` or `end == 0` with non-zero `start`), the loop body never executes. The function returns an empty `[]AssetTransformInput{}` with `nil` error. Back in `export_assets.go`, the empty slice causes the `for _, transformInput := range paymentOps` loop (line 44) to be skipped entirely, and the command proceeds to close the file (line 72), log `0 bytes written` (line 73), and call `MaybeUpload` (line 77) — a complete success-shaped execution with no data.

### Code Paths Examined

- `cmd/export_assets.go:30-34` — branching logic; default path calls `GetPaymentOperations` without captive-core
- `internal/input/assets.go:21-61` — `GetPaymentOperations` creates backend via `CreateLedgerBackend`, no range validation, `for seq := start; seq <= end` loop skips on impossible ranges, returns empty slice + nil error
- `internal/utils/main.go:1011-1041` — `CreateLedgerBackend` creates a datastore or captive-core backend without calling `ValidateLedgerRange`
- `internal/utils/main.go:736-758` — `ValidateLedgerRange` rejects `end == 0`, `end < start`, and future ranges
- `internal/utils/main.go:760-778` — `CreateBackend` (history-archive path) DOES call `ValidateLedgerRange` at line 770
- `internal/input/assets_history_archive.go:13-51` — `GetPaymentOperationsHistoryArchive` uses `CreateBackend`, so the captive-core path IS validated
- `cmd/export_assets.go:44-77` — empty paymentOps causes the processing loop to be skipped; file is closed, bytes logged as 0, upload proceeds

### Findings

The bug is confirmed. `export_assets` on the default datastore path accepts impossible bounded ranges and produces empty success output. The root cause is the same as success entries 015 (ledger/token-transfer) and 016 (transaction-family): `CreateLedgerBackend()` does not call `ValidateLedgerRange()`, and the reader's `for seq := start; seq <= end` loop silently produces an empty result for impossible ranges.

Severity is adjusted from High to Medium because: (1) the root cause is identical to already-confirmed findings for other commands, (2) the impact is limited to asset exports specifically, and (3) the fix is straightforward — add range validation in `GetPaymentOperations()` or at the command level.

The asymmetry between the two code paths is notable: `GetPaymentOperationsHistoryArchive` → `CreateBackend` → `ValidateLedgerRange` (validated), while `GetPaymentOperations` → `CreateLedgerBackend` (not validated). This confirms the bug is backend-specific.

### PoC Guidance

- **Test file**: `cmd/data_integrity_poc_test.go` (append to existing if present, or create)
- **Setup**: Build the binary with `go build ./...`; ensure no captive-core config so the default datastore path is used
- **Steps**:
  1. Call `GetPaymentOperations(100, 50, -1, env, false)` (end < start) and verify it returns empty slice with nil error
  2. Call `GetPaymentOperations(100, 0, -1, env, false)` (end == 0) and verify same
  3. Contrast with `ValidateLedgerRange(100, 50, 1000)` returning an error
- **Assertion**: `GetPaymentOperations` returns `([]AssetTransformInput{}, nil)` for impossible ranges instead of returning an error, confirming the export command would produce an empty success artifact

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestExportAssetsAcceptsImpossibleDatastoreRanges"
**Test Language**: Go

### Demonstration

The test proves that `export_assets` silently accepts impossible bounded ranges (`end < start` and `end == 0`) on the default datastore path, producing empty output files with exit code 0. In contrast, the existing `ValidateLedgerRange()` helper correctly rejects both cases with descriptive errors. This confirms the missing validation gap: `GetPaymentOperations` never calls `ValidateLedgerRange`, allowing impossible ranges to produce success-shaped empty artifacts.

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

// TestExportAssetsAcceptsImpossibleDatastoreRanges demonstrates that the
// datastore-backed export_assets command silently accepts impossible bounded
// ranges (end < start, end == 0) and emits empty success-shaped output
// instead of rejecting them. The existing ValidateLedgerRange helper would
// catch these ranges, but GetPaymentOperations never calls it.
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

### Test Output

```
=== RUN   TestExportAssetsAcceptsImpossibleDatastoreRanges
=== RUN   TestExportAssetsAcceptsImpossibleDatastoreRanges/ValidateLedgerRange_rejects_impossible_ranges
=== RUN   TestExportAssetsAcceptsImpossibleDatastoreRanges/ValidateLedgerRange_rejects_impossible_ranges/end_before_start
=== RUN   TestExportAssetsAcceptsImpossibleDatastoreRanges/ValidateLedgerRange_rejects_impossible_ranges/end_is_zero
=== RUN   TestExportAssetsAcceptsImpossibleDatastoreRanges/export_assets_accepts_end_before_start
=== RUN   TestExportAssetsAcceptsImpossibleDatastoreRanges/export_assets_accepts_end_is_zero
--- PASS: TestExportAssetsAcceptsImpossibleDatastoreRanges (2.52s)
    --- PASS: TestExportAssetsAcceptsImpossibleDatastoreRanges/ValidateLedgerRange_rejects_impossible_ranges (0.00s)
        --- PASS: TestExportAssetsAcceptsImpossibleDatastoreRanges/ValidateLedgerRange_rejects_impossible_ranges/end_before_start (0.00s)
        --- PASS: TestExportAssetsAcceptsImpossibleDatastoreRanges/ValidateLedgerRange_rejects_impossible_ranges/end_is_zero (0.00s)
    --- PASS: TestExportAssetsAcceptsImpossibleDatastoreRanges/export_assets_accepts_end_before_start (1.62s)
    --- PASS: TestExportAssetsAcceptsImpossibleDatastoreRanges/export_assets_accepts_end_is_zero (0.89s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	13.998s
```
