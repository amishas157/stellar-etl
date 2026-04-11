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
