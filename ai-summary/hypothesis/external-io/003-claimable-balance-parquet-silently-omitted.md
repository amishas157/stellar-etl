# H003: `export_ledger_entry_changes --export-balances --write-parquet` silently omits claimable-balance parquet output

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When claimable balances are exported with `--write-parquet`, the Parquet side should contain the same claimable-balance rows that were written to the JSON batch file for that resource.

## Mechanism

`exportTransformedData()` recognizes `transform.ClaimableBalanceOutput` only by setting `skip = true`; it never appends the row to a Parquet slice and never assigns a schema. The command therefore writes correct JSON claimable-balance batches, then silently skips Parquet creation and upload for that resource even though `write-parquet` was requested.

## Trigger

Run `stellar-etl export_ledger_entry_changes --export-balances true --write-parquet ...` on any ledger batch containing claimable-balance changes, such as the existing claimable-balance test range in `cmd/export_ledger_entry_changes_test.go`.

## Target Code

- `cmd/export_ledger_entry_changes.go:90-107` — pre-creates the `claimable_balances` resource bucket when `--export-balances` is enabled
- `cmd/export_ledger_entry_changes.go:331-335` — explicitly marks `ClaimableBalanceOutput` as `skip = true`
- `cmd/export_ledger_entry_changes.go:368-372` — writes/upload Parquet only when `skip` is false
- `README.md:305-316` — advertises `export-balances` as a normal ledger-entry-changes export type

## Evidence

The production switch contains an in-code comment saying claimable-balance parquet is being skipped because it is “not needed in the current scope of work.” There is no compensating warning, alternate artifact, or flag validation preventing users from requesting Parquet for this resource.

## Anti-Evidence

The JSON claimable-balance export still works, so the bug is Parquet-only. If the project intentionally treats claimable-balance Parquet as unsupported, the missing piece may be documentation or flag rejection rather than schema generation, but the current runtime behavior is still silent omission.
