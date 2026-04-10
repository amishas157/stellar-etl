# H004: `export_ledger_entry_changes` leaks JSON file descriptors until later batches stop exporting

**Date**: 2026-04-10
**Subsystem**: external-io
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each resource file opened during `export_ledger_entry_changes` should be closed once its batch has been written and, if configured, uploaded. Long-running or continuous exports should be able to process arbitrarily many batches without exhausting OS file descriptors or dropping later ledger-entry-change outputs.

## Mechanism

`exportTransformedData()` opens a new JSON file with `MustOutFile()` for every resource in every batch, writes records, and never calls `outFile.Close()`. In bounded ranges this may go unnoticed, but in large ranges or `end-ledger=0` streaming mode the command accumulates open descriptors until `os.OpenFile` fails, at which point later batches are never exported.

## Trigger

Run `export_ledger_entry_changes` across a long range or continuously with multiple resource types enabled. After enough batches/resources, the process should hit the host's open-file limit and stop producing later JSON exports even though earlier batches looked healthy.

## Target Code

- `cmd/export_ledger_entry_changes.go:304-372` — opens one `outFile` per resource and never closes it
- `cmd/command_utils.go:31-53` — `MustOutFile()` creates a real OS file descriptor each time

## Evidence

All the other export commands explicitly call `outFile.Close()` before upload or exit, but `exportTransformedData()` does not. The loop is batch-oriented and can run indefinitely, so the missing close compounds over time rather than being a one-off leak.

## Anti-Evidence

Short, bounded exports may finish before hitting the descriptor limit, so the failure is workload-dependent. The bug is still meaningful because it drops later batches rather than corrupting just a single row.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (H001 investigated upload-before-close truncation on the same path but concluded that descriptor leakage, not truncation, was the real issue; H004 is the first hypothesis to directly target the FD leak)

### Trace Summary

`exportTransformedData()` (lines 295–377) is called once per batch from the main `for/select` loop (line 79). Each call iterates `transformedOutput` (line 304) and opens a new `*os.File` via `MustOutFile(path)` (line 311) for each resource type — up to 10 per batch. The `outFile` variable is never closed; it goes out of scope at the end of each loop iteration. All 9 other export commands (`export_operations`, `export_trades`, `export_ledgers`, `export_ledger_transaction`, `export_effects`, `export_assets`, `export_contract_events`, `export_transactions`, `export_token_transfers`) explicitly call `outFile.Close()` before upload. This is a clear pattern deviation.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:79-291` — main `for/select` loop calls `exportTransformedData()` for each batch indefinitely (or until `closeChan`)
- `cmd/export_ledger_entry_changes.go:295-377` — `exportTransformedData()` opens `outFile := MustOutFile(path)` at line 311, writes records in the inner loop (lines 315–366), calls `MaybeUpload` at line 368, but **never calls `outFile.Close()`**
- `cmd/command_utils.go:31-53` — `MustOutFile()` calls `os.OpenFile()` with `O_RDWR|O_CREATE|O_TRUNC`, returning a real OS file descriptor; fatals on error
- `cmd/export_operations.go:30,56` — representative of the 9 other export commands: opens with `MustOutFile`, closes with `outFile.Close()` before upload
- `cmd/export_transactions.go:56`, `cmd/export_trades.go:61`, `cmd/export_ledgers.go:63`, `cmd/export_effects.go:59`, `cmd/export_assets.go:72`, `cmd/export_contract_events.go:57`, `cmd/export_ledger_transaction.go:51`, `cmd/export_token_transfers.go:57` — all 9 close their files explicitly

### Findings

1. **Missing Close() confirmed**: `exportTransformedData()` is the only export path in the entire `cmd/` package that does not call `Close()` on its JSON output file. This is a clear omission relative to the established pattern in all 9 sibling commands.

2. **Accumulation rate**: With up to 10 resource types enabled, each batch leaks up to 10 FDs. In streaming mode (`end-ledger=0`), batches arrive continuously. Go's GC finalizer will eventually reclaim orphaned `*os.File` objects, but finalizer timing is not deterministic — the runtime documentation explicitly states there is no guarantee finalizers run promptly.

3. **Crash behavior**: When the OS FD limit is reached, `os.OpenFile` in `MustOutFile` returns an error, triggering `cmdLogger.Fatal()` (line 49), which terminates the process. All subsequent batches are lost. This is not silent corruption of existing data, but it IS silent data loss — the process stops exporting future batches.

4. **Interaction with upload**: The file is uploaded via `MaybeUpload` (line 368) while the FD is still open. As H001 established, the written bytes are visible to subsequent readers through the filesystem, so the uploaded data is not truncated. The issue is purely about FD exhaustion over time.

### PoC Guidance

- **Test file**: `cmd/export_ledger_entry_changes_test.go` (or create a unit test for `exportTransformedData`)
- **Setup**: Call `exportTransformedData()` in a loop simulating many batches, with a temporary output directory and dummy transform outputs for multiple resource types. Optionally lower the process FD limit with `syscall.Setrlimit(syscall.RLIMIT_NOFILE, ...)` to trigger the issue faster.
- **Steps**: (1) Record the number of open FDs before the loop. (2) Call `exportTransformedData()` N times (e.g., 50 iterations × 10 resource types = 500 FDs). (3) Record the number of open FDs after the loop.
- **Assertion**: The number of open FDs after the loop should be significantly higher than before (proportional to N × resource_count), demonstrating that descriptors are not being released. Alternatively, with a low FD limit, assert that `MustOutFile` eventually fatals.
