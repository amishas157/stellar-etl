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
