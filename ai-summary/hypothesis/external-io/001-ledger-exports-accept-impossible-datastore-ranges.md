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
