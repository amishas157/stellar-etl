# H004: `ExportEntry()` hides JSON write failures behind success-shaped returns

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If writing an exported JSON row fails, the export command should receive a non-nil error, count the row as failed, and avoid reporting a successful export of a partially written file.

## Mechanism

`ExportEntry()` logs errors from `outFile.Write()` and `outFile.WriteString("\n")` but always returns `numBytes + newLineNumBytes, nil`. Every JSON export command relies on that error return to decide whether to increment failure counts and whether the output file is valid enough to upload, so partial writes can silently leave truncated JSON lines or missing newlines while the CLI reports success and proceeds to `MaybeUpload()`.

## Trigger

Run any JSON export (`export_transactions`, `export_ledgers`, `export_ledger_entry_changes`, etc.) on a filesystem that returns a short write or write error mid-export, such as a full disk or unstable network mount. The command should fail that row, but this code path should keep going and may upload a malformed line-delimited JSON file.

## Target Code

- `cmd/command_utils.go:55-86` - logs `Write`/`WriteString` errors but returns `nil`
- `cmd/export_ledgers.go:50-73` - representative caller that trusts `ExportEntry()` errors for failure accounting
- `cmd/export_transactions.go:43-66` - representative caller that trusts `ExportEntry()` errors for failure accounting
- `cmd/export_ledger_entry_changes.go:311-372` - batch exporter also treats a nil `ExportEntry()` return as success

## Evidence

The only non-nil return in `ExportEntry()` is the later `json.Marshal(i)` failure path. Actual file I/O failures at lines 78-85 are merely logged, after which the function still returns success to all callers.

## Anti-Evidence

Pure marshalling failures are surfaced correctly because the second `json.Marshal(i)` return is propagated. On healthy local disks, the bug stays latent because `Write` and `WriteString` usually succeed.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`ExportEntry()` at `cmd/command_utils.go:55-86` performs two file I/O operations: `outFile.Write(marshalled)` at line 78 and `outFile.WriteString("\n")` at line 82. Both capture the error return, log it via `cmdLogger.Errorf`, then fall through to line 86 which unconditionally returns `nil`. All 10 export commands (ledgers, transactions, operations, effects, trades, assets, contract_events, ledger_entry_changes, token_transfers, ledger_transaction) call `ExportEntry()` and check its error return to decide whether to increment `numFailures` and whether to proceed with `MaybeUpload()`. Since I/O errors never surface, callers always treat writes as successful.

### Code Paths Examined

- `cmd/command_utils.go:ExportEntry:78-86` — `outFile.Write()` error captured at line 79, logged at line 80, then execution falls through; `outFile.WriteString("\n")` error captured at line 83, logged at line 84, then execution falls through; line 86 returns `numBytes + newLineNumBytes, nil` unconditionally
- `cmd/export_ledgers.go:50-55` — checks `err` from `ExportEntry()`, increments `numFailures` and `continue`s on error; since error is always nil, failure count is never incremented for I/O failures
- `cmd/export_transactions.go:43-48` — identical pattern: error check gates `numFailures` increment
- `cmd/export_operations.go:43` — same pattern
- `cmd/export_effects.go:45` — same pattern
- `cmd/export_trades.go:47` — same pattern
- `cmd/export_assets.go:59` — same pattern
- `cmd/export_token_transfers.go:47` — same pattern
- `cmd/export_contract_events.go:43` — same pattern
- `cmd/export_ledger_transaction.go:42` — same pattern
- `cmd/export_ledger_entry_changes.go:316-319` — returns error directly to caller; since nil, batch loop continues and upload proceeds

### Findings

The bug is confirmed. `ExportEntry()` has a function signature `(int, error)` but only returns a non-nil error from the second `json.Marshal()` call (line 73-76). The two file I/O operations at lines 78-85 both capture errors but only log them. This creates two concrete failure modes:

1. **Truncated JSON rows**: If `outFile.Write(marshalled)` partially writes (short write) or fails entirely, the output file contains an incomplete or missing JSON line. The caller does not know and does not increment its failure counter.

2. **Missing newlines**: If `outFile.WriteString("\n")` fails after a successful `Write`, the file contains concatenated JSON objects without line delimiters, making downstream line-delimited JSON parsers misparse multiple rows.

In both cases, `MaybeUpload()` is subsequently called on the malformed file, and `PrintTransformStats()` reports 0 failures for what may be many corrupted rows. The upload to GCS succeeds with bad data.

### PoC Guidance

- **Test file**: `cmd/command_utils_test.go` (create if not present; or append to an existing cmd-level test file)
- **Setup**: Create a mock `*os.File` or use a real file on a filesystem where writes can be made to fail. The simplest approach: create an `os.File` wrapper that returns `os.ErrClosed` from `Write`/`WriteString`, or close the file handle before calling `ExportEntry()`.
- **Steps**:
  1. Create a valid transform output struct (e.g., `transform.LedgerOutput{}`)
  2. Open a file with `MustOutFile()`, then close it immediately (`outFile.Close()`)
  3. Call `ExportEntry(transformed, outFile, nil)`
  4. Observe the returned error
- **Assertion**: Assert that the returned `error` is non-nil when `Write` fails. Currently it will be `nil`, demonstrating the bug. Also assert that the returned `numBytes` does not include bytes from the failed write.
