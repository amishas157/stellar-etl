# H001: Bounded change exports accept `end < start` and finish with no batches

**Date**: 2026-04-15
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes` is run in bounded mode, an impossible range such as `--start-ledger 100 --end-ledger 50` should be rejected before the export starts. The command should not create a success-shaped empty export folder for a range that can never correspond to real ledger history.

## Mechanism

`export_ledger_entry_changes` never applies `utils.ValidateLedgerRange()` before calling `backend.PrepareRange(BoundedRange(start, end))` and then handing the same values to `StreamChanges()`. On the default datastore backend, `PrepareRange()` accepts the malformed range, and `StreamChanges()` computes `batchEnd = min(start+batchSize, end)`, which is immediately less than `batchStart`; the loop never runs, the channel closes normally, and the command exits as if the requested range simply contained no changes.

## Trigger

1. Run `stellar-etl export_ledger_entry_changes --start-ledger 100 --end-ledger 50 --output <dir>`.
2. Observe that the command creates the output folders and exits through the normal completion path instead of rejecting the request.
3. The correct behavior is a validation error, because no bounded ledger range can satisfy `start > end`.

## Target Code

- `cmd/export_ledger_entry_changes.go:61-78` — prepares a bounded backend range and launches `StreamChanges()` without validating bounded inputs
- `internal/input/changes.go:163-177` — computes `batchEnd` from the invalid `end` value, skips the batch loop when `batchStart < batchEnd` is false, then closes the channel normally
- `internal/utils/main.go:736-758` — existing shared validator that already rejects `end < start`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledgerbackend/buffered_storage_backend.go:172-189` — datastore `PrepareRange()` records the malformed bounded range without rejecting it

## Evidence

The local command path performs no validation between flag parsing and `PrepareRange()`. The datastore backend's `PrepareRange()` implementation simply records the supplied `Range`, and `StreamChanges()` uses only arithmetic on `start` and `end` to decide whether any batch exists; for `start > end`, it closes the stream without ever calling `ExtractBatch()`.

## Anti-Evidence

The unsupported unbounded mode (`end == 0`) fails differently and is already documented as unsupported, so this hypothesis is intentionally narrower: a malformed bounded range with `end < start`. If some callers intentionally relied on "empty output means bad range," that would weaken the finding, but the rest of the codebase already carries an explicit validator for this exact condition, which points the other way.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-15
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from CLI flag parsing through `MustCoreFlags`/`MustCommonFlags` → `CreateLedgerBackend` (datastore path) → `PrepareRange(BoundedRange(startNum, endNum))` → `StreamChanges(start=100, end=50, batchSize=64)`. Confirmed that no validation exists between flag parsing and `PrepareRange` on the datastore backend path. In contrast, the captive-core path in `PrepareCaptiveCore()` (changes.go:61-69) explicitly calls `ValidateLedgerRange(start, end, latest)` which rejects `end < start`. The `StreamChanges` loop guard `batchStart < batchEnd` evaluates `100 < 50 → false`, so zero batches are emitted and the command exits with success status.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:31-78` — command handler: parses flags, creates datastore backend, calls `PrepareRange(BoundedRange(startNum, commonArgs.EndNum))` with no range validation
- `internal/utils/main.go:460-525` — `MustCommonFlags`: parses `--end-ledger` as raw uint32, no cross-validation with start
- `internal/utils/main.go:596-628` — `MustCoreFlags`: parses `--start-ledger` as raw uint32, no cross-validation with end
- `internal/utils/main.go:1009-1041` — `CreateLedgerBackend`: creates buffered storage backend with no range validation
- `internal/input/changes.go:162-178` — `StreamChanges`: computes `batchEnd = min(start+batchSize, end)` = `min(164, 50)` = 50; loop guard `100 < 50` is false; zero iterations; channel closes normally
- `internal/input/changes.go:33-80` — `PrepareCaptiveCore`: the captive-core path DOES call `ValidateLedgerRange(start, end, latest)` at line 68, confirming the intent to validate ranges
- `internal/utils/main.go:735-758` — `ValidateLedgerRange`: existing validator that explicitly rejects `end < start` at line 745-746

### Findings

The hypothesis is correct. The `export_ledger_entry_changes` command on the default datastore backend path skips range validation that the captive-core path already performs. With `end < start`:

1. `PrepareRange(BoundedRange(100, 50))` succeeds (datastore backend records the range without validation)
2. `StreamChanges(start=100, end=50)` computes `batchEnd = min(164, 50) = 50`, loop guard `100 < 50` is false
3. Zero batches emitted, channel closes, command exits with success status
4. Output directories are created (lines 43-51) but contain no export files
5. No error is logged or returned

The severity is downgraded from High to Medium because this is an operational correctness issue (missing input validation that masks a configuration error) rather than structural data corruption. The command produces no data rather than wrong data. However, in automated pipeline contexts, a success exit code with empty output could cause downstream systems to incorrectly conclude that no ledger entry changes occurred in the requested range.

The `ValidateLedgerRange()` function at `utils/main.go:736` already implements the exact check needed (`end < start` at line 745), and the captive-core path at `changes.go:68` already calls it. The datastore path simply omits this validation.

This is distinct from success finding 003 (single-ledger `start == end` range skips loop — a valid range that produces no batches due to a loop guard bug) and success finding 008 (continuous mode `BoundedRange(start, 0)` — a different malformed range specific to unbounded mode).

### PoC Guidance

- **Test file**: `internal/input/changes_test.go` (or append to `internal/input/data_integrity_poc_test.go` if it exists)
- **Setup**: Use the existing `mockExtractBatch` pattern from success finding 003's PoC. Override `ExtractBatch` with a mock that returns an empty `ChangeBatch`.
- **Steps**: Call `StreamChanges(nil, 100, 50, 64, changeChan, closeChan, env, logger)` — a bounded range where start > end.
- **Assertion**: Assert that `changeChan` receives zero batches AND that the function should have returned an error or panic (demonstrating the missing validation). Alternatively, test at the command level by verifying that `export_ledger_entry_changes --start-ledger 100 --end-ledger 50` exits with a non-zero status code. The current behavior (zero batches, success exit) is the bug.
