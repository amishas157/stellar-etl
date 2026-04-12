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
