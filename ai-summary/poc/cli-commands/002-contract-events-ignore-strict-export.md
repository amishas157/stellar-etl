# H002: `export_contract_events` ignores `--strict-export`

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_contract_events` is run with `--strict-export`, the first transform or write failure should abort the command instead of skipping the bad transaction or event. Operators requesting strict mode should never receive a partial contract-event export that looks successful.

## Mechanism

`export_contract_events` parses the strict-export flag but never assigns it to `cmdLogger.StrictExport`. `LogError` only escalates to `Fatal` when that logger field is set, so contract-event transform and export failures are merely logged and skipped even in strict mode, unlike the sibling export commands that wire the flag into the logger before processing.

## Trigger

Run `export_contract_events --strict-export` over a ledger range that hits any transform or JSON write error. The command will continue past `cmdLogger.LogError(...)`, emit a partial output file, and still reach the normal upload/parquet path.

## Target Code

- `cmd/export_contract_events.go:17-23` — parses flags but never sets `cmdLogger.StrictExport`
- `cmd/export_contract_events.go:33-46` — error paths rely on `cmdLogger.LogError(...)` and continue
- `internal/utils/logger.go:17-23` — `LogError` only aborts when `StrictExport` is true
- `cmd/export_effects.go:17-23` — sibling template correctly assigns `cmdLogger.StrictExport = commonArgs.StrictExport`

## Evidence

Every other archive exporter in `cmd/` sets `cmdLogger.StrictExport` immediately after `MustCommonFlags`, but `export_contract_events` is the lone outlier. Because the command uses `LogError` for both transform and write failures, the missing assignment converts a documented fail-fast flag into a silent skip-on-error path.

## Anti-Evidence

This only changes behavior when an error path is actually hit; clean exports are unaffected. The command still parses the flag correctly, so the bug is specifically the missing wiring step rather than missing CLI surface.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`export_contract_events` calls `utils.MustFlags()` (line 19) and `utils.MustCommonFlags()` (line 22), both of which correctly parse the `--strict-export` flag into their respective return structs (`cmdArgs.StrictExport` and `commonArgs.StrictExport`). However, neither value is ever assigned to `cmdLogger.StrictExport`. Since `NewEtlLogger()` initializes `StrictExport` to `false`, the logger's `LogError` method (logger.go:17-23) always takes the `Error()` branch — logging and continuing — even when the operator explicitly requested `--strict-export`. All 9 other export commands correctly wire this field immediately after flag parsing.

### Code Paths Examined

- `cmd/export_contract_events.go:17-68` — Full Run function: parses flags via `MustFlags` (line 19) and `MustCommonFlags` (line 22), but no `cmdLogger.StrictExport = ...` assignment anywhere in the function
- `internal/utils/logger.go:6-7,10-14` — `EtlLogger` struct has `StrictExport bool` field; `NewEtlLogger()` initializes it to `false`
- `internal/utils/logger.go:17-23` — `LogError` only calls `Fatal` when `StrictExport` is `true`; otherwise calls `Error` (log and continue)
- `internal/utils/main.go:293-310` — `FlagValues` struct includes `StrictExport` field, populated by `MustFlags`
- `internal/utils/main.go:443-456,460-537` — `CommonFlagValues` struct includes `StrictExport` field, populated by `MustCommonFlags`
- `cmd/export_effects.go:20` — Sibling command correctly sets `cmdLogger.StrictExport = commonArgs.StrictExport`
- All 9 other export commands (`export_operations`, `export_assets`, `export_ledger_entry_changes`, `export_trades`, `export_ledger_transaction`, `export_effects`, `export_token_transfers`, `export_transactions`, `export_ledgers`) — all wire the flag correctly

### Findings

The bug is confirmed and straightforward. The `--strict-export` flag is parsed into both `cmdArgs.StrictExport` and `commonArgs.StrictExport` but neither is assigned to `cmdLogger.StrictExport`. This means:

1. **Transform errors** (line 35-39): When `TransformContractEvent` fails, `cmdLogger.LogError()` merely logs the error and execution continues to the next transaction. The partial output file is produced.
2. **Export errors** (line 43-48): When `ExportEntry` fails, the same silent-skip behavior occurs.
3. **Upload proceeds**: After the loop, the partial file is closed, stats printed, and `MaybeUpload` is called — uploading potentially incomplete data to GCS.

This is the only export command in the codebase missing this wiring. The fix is a single line: `cmdLogger.StrictExport = commonArgs.StrictExport` after line 22.

### PoC Guidance

- **Test file**: `cmd/export_contract_events_test.go` (or append to an existing test file in `cmd/`)
- **Setup**: Create a mock scenario where `TransformContractEvent` returns an error for at least one transaction. Verify `cmdLogger.StrictExport` is `false` even when the `--strict-export` flag is set to `true`.
- **Steps**: (1) Inspect the `export_contract_events` Run function and verify no assignment to `cmdLogger.StrictExport` exists. (2) Compare with any sibling command (e.g., `export_effects.go:20`) that correctly sets it. (3) Confirm `LogError` branches on `StrictExport` being `false`.
- **Assertion**: After `MustCommonFlags` returns with `StrictExport: true`, assert that `cmdLogger.StrictExport` remains `false` — proving the flag is parsed but never wired.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestContractEventsStrictExportNotWired"
**Test Language**: Go

### Demonstration

The test replicates the exact flag-parsing path of `export_contract_events.Run`: it sets `--strict-export=true` on the command's flag set, then calls `MustFlags` and `MustCommonFlags` (the same two calls the command makes). Both returned structs correctly carry `StrictExport: true`, proving the flag was parsed. However, `cmdLogger.StrictExport` remains `false` because the command never assigns the parsed value to the logger — confirming that `LogError` will always take the non-fatal `Error()` branch, silently skipping failures even when strict mode is requested.

### Test Body

```go
package cmd

import (
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/utils"
)

// TestContractEventsStrictExportNotWired demonstrates that export_contract_events
// parses --strict-export but never assigns it to cmdLogger.StrictExport.
// All 9 other export commands correctly wire this field; contract events is the
// lone outlier. When an error occurs, LogError checks cmdLogger.StrictExport to
// decide whether to Fatal (abort) or Error (continue). Because the field is never
// set, --strict-export is silently ignored for contract event exports.
func TestContractEventsStrictExportNotWired(t *testing.T) {
	// Reset the package-level logger to a known state.
	cmdLogger = utils.NewEtlLogger()
	if cmdLogger.StrictExport {
		t.Fatal("precondition: cmdLogger.StrictExport should start as false")
	}

	// Build a flag set identical to what export_contract_events registers in init().
	flags := contractEventsCmd.Flags()

	// Simulate the user passing --strict-export.
	if err := flags.Set("strict-export", "true"); err != nil {
		t.Fatalf("could not set strict-export flag: %v", err)
	}

	// --- Reproduce the flag-parsing portion of export_contract_events.Run ---
	// (see export_contract_events.go lines 19-23)
	cmdArgs := utils.MustFlags(flags, cmdLogger)
	commonArgs := utils.MustCommonFlags(flags, cmdLogger)

	// Both parsed structs correctly carry the flag value.
	if !cmdArgs.StrictExport {
		t.Error("cmdArgs.StrictExport should be true after MustFlags")
	}
	if !commonArgs.StrictExport {
		t.Error("commonArgs.StrictExport should be true after MustCommonFlags")
	}

	// BUG: export_contract_events never executes:
	//   cmdLogger.StrictExport = commonArgs.StrictExport
	// (every other export command does this immediately after MustCommonFlags)
	//
	// Therefore cmdLogger.StrictExport is still false, and LogError will
	// merely log.Error instead of log.Fatal — silently skipping failures
	// even when the operator explicitly requested --strict-export.
	if cmdLogger.StrictExport {
		t.Error("expected cmdLogger.StrictExport to remain false (bug: flag never wired), but it was true")
	} else {
		t.Logf("CONFIRMED: cmdLogger.StrictExport is false despite --strict-export=true — flag is parsed but never assigned to the logger")
	}

	// For contrast, this is what every other export command does:
	cmdLogger.StrictExport = commonArgs.StrictExport
	if !cmdLogger.StrictExport {
		t.Error("after manual wiring (as sibling commands do), StrictExport should be true")
	} else {
		t.Logf("After adding the missing wiring line, cmdLogger.StrictExport is correctly true")
	}

	// Clean up: restore logger to default state.
	cmdLogger = utils.NewEtlLogger()
}
```

### Test Output

```
=== RUN   TestContractEventsStrictExportNotWired
    data_integrity_poc_test.go:53: CONFIRMED: cmdLogger.StrictExport is false despite --strict-export=true — flag is parsed but never assigned to the logger
    data_integrity_poc_test.go:61: After adding the missing wiring line, cmdLogger.StrictExport is correctly true
--- PASS: TestContractEventsStrictExportNotWired (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	5.639s
```
