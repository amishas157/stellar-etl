# H004: claimable balance parquet is silently skipped in ledger-entry-change exports

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: Requested parquet subset silently missing
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes` is run with `--export-balances` and `--write-parquet`, each batch that emits claimable balance JSON rows should either also emit a `claimable_balances` parquet file or fail clearly before starting the export. A successful run should not quietly omit the parquet sidecar for that resource.

## Mechanism

`exportTransformedData()` recognizes `transform.ClaimableBalanceOutput` only to set `skip = true` with a comment that parquet conversion is being skipped. Because `write-parquet` is a global command flag, the exporter still writes the JSON file and completes successfully, but it suppresses `WriteParquet(...)` for the claimable-balances resource without surfacing any unsupported-output error.

## Trigger

Run `export_ledger_entry_changes --export-balances true --write-parquet` on any ledger range that contains claimable balance changes.

## Target Code

- `cmd/export_ledger_entry_changes.go:160-172` — claimable balance rows are transformed and queued for export
- `cmd/export_ledger_entry_changes.go:321-335` — `ClaimableBalanceOutput` forces `skip = true`
- `cmd/export_ledger_entry_changes.go:368-373` — the batch still uploads/writes JSON and quietly suppresses parquet when `skip` is true
- `internal/utils/main.go:231-246` — `write-parquet` is a common flag exposed on this command

## Evidence

The code comment explicitly states that `ClaimableBalanceOutputParquet` is being skipped because it is "not needed in the current scope of work," which means the omission is deliberate but user-invisible. The command test suite exercises claimable-balance JSON exports, so this is a reachable production code path rather than dead code.

## Anti-Evidence

The absence of a `ClaimableBalanceOutputParquet` type shows the feature is incomplete, not accidentally half-mapped. But because the command still accepts `--write-parquet` and does not reject the unsupported resource, the observed behavior is silent data loss from the caller's perspective.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of ai-summary/fail/cli-commands/001-claimable-balance-parquet-skip-is-intentional.md
**Failed At**: reviewer

### Trace Summary

Traced the `exportTransformedData()` function in `cmd/export_ledger_entry_changes.go:295-377`. Confirmed that the type switch at line 331 explicitly handles `transform.ClaimableBalanceOutput` by setting `skip = true`, which prevents `WriteParquet()` from being called at line 370. The `ClaimableBalanceOutputParquet` type is commented out in `internal/transform/schema_parquet.go:131-134` with an explanatory note about nested struct handling. This is the same code path and finding already investigated under a different subsystem.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:exportTransformedData:304-376` — confirmed `skip = true` for `ClaimableBalanceOutput` at line 335, parquet gated by `!skip` at line 370
- `internal/transform/schema_parquet.go:131-134` — `ClaimableBalanceOutputParquet` is commented out with a "not needed in current scope" explanation
- `ai-summary/fail/cli-commands/001-claimable-balance-parquet-skip-is-intentional.md` — prior investigation of the identical finding

### Why It Failed

This is a duplicate of a previously investigated hypothesis (`ai-summary/fail/cli-commands/001-claimable-balance-parquet-skip-is-intentional.md`) which already concluded that the `skip = true` guard for `ClaimableBalanceOutput` is an intentional feature gap, not a data corruption bug. The code explicitly handles this case with comments explaining the design decision. While the silent skip without a user-facing warning is a UX gap, it does not constitute data corruption — JSON output is still produced correctly, and the parquet omission is a deliberate scope boundary due to nested struct complexity.

### Lesson Learned

When evaluating "silent skip" hypotheses, check fail directories across all subsystems since the same code path may have been investigated under a different subsystem classification (in this case `cli-commands` vs `export-pipeline`).
