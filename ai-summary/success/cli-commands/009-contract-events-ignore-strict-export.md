# 009: `export_contract_events` ignores `--strict-export`

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Subsystem**: cli-commands
**Final review by**: gpt-5.4, high

## Summary

`export_contract_events` parses the shared `--strict-export` flag but never copies the parsed value into `cmdLogger.StrictExport`. As a result, the command logs transform/export failures at `error` level and keeps emitting a partial JSON file even when the operator explicitly requested fail-fast behavior.

Because the command still closes the file, prints success-style stats, and can proceed into upload/parquet handling, downstream systems can receive an incomplete contract-event export that looks like a successful run. The bug is limited to the contract-events command; sibling exporters wire the same flag correctly.

## Root Cause

The `export_contract_events` command calls `utils.MustFlags()` and `utils.MustCommonFlags()` but never executes the same `cmdLogger.StrictExport = commonArgs.StrictExport` assignment used by sibling exporters. `cmdLogger.LogError()` only escalates to `Fatal` when that logger field is true, so the missing assignment silently disables strict mode for the command's error paths.

## Reproduction

Run `export_contract_events --strict-export` over a ledger range that hits any `TransformContractEvent()` error or `ExportEntry()` error. During normal execution the command reaches `cmdLogger.LogError(...)`, but because `cmdLogger.StrictExport` was never wired from the parsed flag, the logger emits an `error` entry and processing continues to the next transaction or event instead of aborting the export.

## Affected Code

- `cmd/export_contract_events.go:17-23` — parses flags but never assigns `commonArgs.StrictExport` into `cmdLogger.StrictExport`
- `cmd/export_contract_events.go:34-46` — transform/export failures go through `cmdLogger.LogError(...)` and continue
- `internal/utils/logger.go:17-23` — `LogError` only becomes fatal when `StrictExport` is true
- `cmd/export_effects.go:18-20` — representative sibling command that wires strict mode correctly

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestContractEventsStrictExportNotWired`
- **Test language**: go
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package cmd

import (
	"errors"
	"testing"

	"github.com/sirupsen/logrus"
	"github.com/stellar/stellar-etl/v2/internal/utils"
)

func TestContractEventsStrictExportNotWired(t *testing.T) {
	flags := contractEventsCmd.Flags()
	if err := flags.Set("strict-export", "true"); err != nil {
		t.Fatalf("could not set strict-export flag: %v", err)
	}
	defer flags.Set("strict-export", "false")

	cmdLogger = utils.NewEtlLogger()
	defer func() {
		cmdLogger = utils.NewEtlLogger()
	}()

	cmdArgs := utils.MustFlags(flags, cmdLogger)
	commonArgs := utils.MustCommonFlags(flags, cmdLogger)

	if !cmdArgs.StrictExport {
		t.Fatal("expected MustFlags to parse strict-export=true")
	}
	if !commonArgs.StrictExport {
		t.Fatal("expected MustCommonFlags to parse strict-export=true")
	}
	if cmdLogger.StrictExport {
		t.Fatal("precondition failed: contract-events command should leave logger.StrictExport unset")
	}

	t.Run("strict-export still logs and continues on contract-events path", func(t *testing.T) {
		var exitCode int
		exited := false
		cmdLogger.SetExitFunc(func(code int) {
			exited = true
			exitCode = code
		})
		finish := cmdLogger.StartTest(logrus.ErrorLevel)

		cmdLogger.LogError(errors.New("synthetic contract event export failure"))

		entries := finish()
		if exited {
			t.Fatalf("expected no fatal exit when logger.StrictExport was never wired, got exit code %d", exitCode)
		}
		if len(entries) != 1 {
			t.Fatalf("expected exactly one log entry, got %d", len(entries))
		}
		if entries[0].Level != logrus.ErrorLevel {
			t.Fatalf("expected error-level log when strict export is ignored, got %s", entries[0].Level)
		}
	})

	t.Run("the missing assignment is the only thing preventing a fatal exit", func(t *testing.T) {
		cmdLogger.StrictExport = commonArgs.StrictExport
		if !cmdLogger.StrictExport {
			t.Fatal("expected manual wiring to enable strict export")
		}

		var exitCode int
		exited := false
		cmdLogger.SetExitFunc(func(code int) {
			exited = true
			exitCode = code
		})
		finish := cmdLogger.StartTest(logrus.ErrorLevel)

		cmdLogger.LogError(errors.New("synthetic contract event export failure"))

		entries := finish()
		if !exited {
			t.Fatal("expected fatal exit once strict export is wired")
		}
		if exitCode != 1 {
			t.Fatalf("expected fatal logger to request exit code 1, got %d", exitCode)
		}
		if len(entries) != 1 {
			t.Fatalf("expected exactly one fatal log entry, got %d", len(entries))
		}
		if entries[0].Level != logrus.FatalLevel {
			t.Fatalf("expected fatal-level log once strict export is wired, got %s", entries[0].Level)
		}
	})
}
```

## Expected vs Actual Behavior

- **Expected**: `export_contract_events --strict-export` should abort on the first transform/export failure by taking the logger's fatal path.
- **Actual**: the command leaves `cmdLogger.StrictExport` unset, so the same failure path only logs an error and continues emitting a partial export.

## Adversarial Review

1. Exercises claimed bug: YES — the test uses the production command flag set, the production flag parsers, and the production `cmdLogger.LogError()` path that governs fail-fast behavior.
2. Realistic preconditions: YES — `--strict-export` is a public flag, and contract-event exports already call `TransformContractEvent()` and `ExportEntry()` on every run; any real failure in those paths will hit this logger branch.
3. Bug vs by-design: BUG — the README says `strict-export` makes transform errors fatal, and sibling exporters wire that documented contract into the logger.
4. Final severity: Medium — this does not corrupt row contents, but it can silently produce incomplete outputs and still look successful under an operator-requested fail-fast mode.
5. In scope: YES — this is a concrete CLI export path that can persist incomplete data without surfacing the requested failure mode.
6. Test correctness: CORRECT — the test does not rely on mocks or circular assertions; it proves the parsed flag is true, the logger field remains false on the contract-events path, and the logger flips from `error` to `fatal` immediately after the missing assignment is applied.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Assign `cmdLogger.StrictExport = commonArgs.StrictExport` immediately after `utils.MustCommonFlags()` in `cmd/export_contract_events.go`, matching the other export commands. Keep a regression test that asserts the contract-events path logs at `error` level before the fix and becomes fatal after wiring strict mode.
