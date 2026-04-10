# H002: ExportEntry counts failed JSON writes as successful exports

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a transformed row cannot be written to the JSON output file, the export command should receive a non-nil error, count that row as failed, and avoid reporting or uploading a silently truncated export. The output file should either contain every successfully transformed row or the command should fail explicitly.

## Mechanism

`ExportEntry` logs errors from `json.Marshal`, `decoder.Decode`, `outFile.Write`, and `outFile.WriteString`, but it only returns a non-nil error for the final `json.Marshal(i)` call. Every caller in `cmd/` treats a nil return as success, so disk-full, short-write, or newline-write failures can produce missing or malformed rows while transform stats and follow-on upload logic still treat the export as successful.

## Trigger

1. Start any JSON export command that writes many rows, such as `export_transactions`, `export_operations`, `export_effects`, or `export_ledger_entry_changes`.
2. Hit a legitimate filesystem error mid-run, such as `ENOSPC` or `EIO`, during `outFile.Write` or `outFile.WriteString`.
3. The command logs an error but still increments byte counters, keeps processing later rows, and can upload a truncated file as if the batch were complete.

## Target Code

- `cmd/command_utils.go:ExportEntry:55-86` — logs write and decode errors but returns `nil`
- `cmd/export_transactions.go:Run:43-49` — treats `ExportEntry` nil as a successful row
- `cmd/export_operations.go:Run:43-49` — same success accounting pattern
- `cmd/export_effects.go:Run:45-51` — same success accounting pattern
- `cmd/export_ledger_entry_changes.go:exportTransformedData:315-319` — same success accounting pattern before upload

## Evidence

The write path has three separate logged error branches (`json.Marshal(entry)`, `outFile.Write`, `outFile.WriteString`) that do not change the returned error. All callers rely on the returned error to increment `numFailures` or abort the batch, so these failures are observationally downgraded from "failed export" to "successful export with a log line."

## Anti-Evidence

The function does correctly fail if the second `json.Marshal(i)` call errors, and several callers are prepared to handle a non-nil return. The silent corruption only happens on the broader set of I/O and decode failures that `ExportEntry` currently suppresses.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`ExportEntry` (command_utils.go:55-86) performs four fallible operations: initial `json.Marshal`, `decoder.Decode`, `outFile.Write`, and `outFile.WriteString`. Only the second `json.Marshal` (line 73-76) returns its error to the caller. The other three operations log errors via `cmdLogger.Errorf` but allow execution to continue, ultimately returning `nil` on line 86. All 9 export commands that call `ExportEntry` use the returned error to decide whether to increment `numFailures` or count the row as successful, meaning write failures are silently counted as successes.

### Code Paths Examined

- `cmd/command_utils.go:ExportEntry:55-86` — Confirmed: lines 59-60 log first marshal error but don't return; lines 66-68 log decode error but don't return; lines 79-81 log `outFile.Write` error but don't return; lines 83-85 log `outFile.WriteString` error but don't return. Line 86 unconditionally returns `numBytes + newLineNumBytes, nil`.
- `cmd/export_transactions.go:43-49` — Confirmed: checks `err != nil` from `ExportEntry`, increments `numFailures` only on non-nil error. Since write errors return nil, failed writes are counted as successes and `totalNumBytes` accumulates the (possibly 0) byte count.
- `cmd/export_operations.go:43-49` — Same pattern as transactions.
- `cmd/export_ledger_entry_changes.go:315-319` — Same pattern; additionally, after the loop, the command proceeds to `MaybeUpload` which uploads the potentially truncated file to GCS.

### Findings

The bug is confirmed and real. There are four distinct error-swallowing sites in `ExportEntry`:

1. **First `json.Marshal` failure (line 57-60)**: If the initial marshal fails, `m` is nil. The code then creates a decoder over `bytes.NewReader(nil)`, which yields an empty reader. The subsequent `Decode` will also fail, logging a second error. The `extra` map is then merged into an empty map, and the second `json.Marshal` on line 73 produces `{}` plus any extra fields — a malformed row that silently replaces the intended data.

2. **`decoder.Decode` failure (line 64-68)**: If decode fails but marshal succeeded, `i` remains an empty map. Same result as above — a near-empty JSON object is written.

3. **`outFile.Write` failure (line 78-81)**: If the file write fails (ENOSPC, EIO, broken pipe), `numBytes` may be 0 or a short count, but the function returns nil. The caller counts this as a successful export and adds the byte count to its total.

4. **`outFile.WriteString("\n")` failure (line 82-85)**: If the newline write fails, the next row's JSON will be concatenated with the current row on the same line, producing malformed JSONL that downstream parsers will fail to read correctly.

The downstream consequences are:
- `PrintTransformStats` reports inflated success counts
- `MaybeUpload` uploads truncated/malformed files to GCS
- Downstream BigQuery or analytics ingestion receives incomplete data without any signal that the export was partial

### PoC Guidance

- **Test file**: `cmd/command_utils_test.go` (create if needed, or append to existing test file in `cmd/`)
- **Setup**: Create a mock `*os.File` by opening a file on a read-only filesystem or use a pipe that is closed before writing. Alternatively, use `os.Pipe()` and close the write end immediately.
- **Steps**:
  1. Create a valid transform output struct (e.g., `transform.LedgerOutput{}` with some fields set)
  2. Create a broken writer — e.g., `r, w := os.Pipe(); w.Close()` then pass `r` (the read end, which will error on Write)
  3. Call `ExportEntry(output, brokenFile, nil)`
  4. Observe the returned error
- **Assertion**: Assert that `ExportEntry` returns a non-nil error when `outFile.Write` or `outFile.WriteString` fails. Currently it returns nil, confirming the bug. A secondary assertion: verify that when the first `json.Marshal` fails, the function returns an error rather than writing a near-empty JSON object.
