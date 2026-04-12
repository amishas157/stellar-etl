# H003: Asset export treats impossible ranges as empty asset sets

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_assets` should reject bounded requests like `--start-ledger 100 --end-ledger 50` or `--end-ledger 0`, because those ranges cannot correspond to a valid Stellar ledger interval and an empty asset file is materially different from a failed export.

## Mechanism

On the default path, `export_assets` calls `input.GetPaymentOperations()`, which prepares a bounded datastore range but never validates the start/end pair. When `seq := start; seq <= end` is false at entry, the helper returns an empty operation slice, and `export_assets` then emits a zero-row asset file as if the requested interval legitimately contained no relevant operations.

## Trigger

1. Run `stellar-etl export_assets --start-ledger 100 --end-ledger 50 -o assets.json`.
2. Because no validation fires before the loop, the command may succeed with an empty asset export instead of rejecting the impossible range.

## Target Code

- `internal/input/assets.go:GetPaymentOperations:21-60` — missing `ValidateLedgerRange()` before iterating the bounded range
- `cmd/export_assets.go:Run:17-82` — accepts an empty `paymentOps` slice and produces a zero-row asset export
- `internal/utils/main.go:735-758` — contains the validation helper that this path currently skips

## Evidence

The asset reader follows the same datastore-backed shape as `GetLedgers()` but lacks the corresponding history-archive validation guard. `export_assets` then deduplicates and writes rows only if `paymentOps` is non-empty, so an impossible range becomes indistinguishable from "no new assets in this interval".

## Anti-Evidence

When `--captive-core` is enabled, `export_assets` switches to `GetPaymentOperationsHistoryArchive()`, which likely inherits the validating `CreateBackend()` path instead of the silent datastore path. That suggests the bug may be configuration-specific rather than universal across all asset exports.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated. H001 covers `GetLedgers()` for `export_ledgers`/`export_token_transfers`; H002 covers `GetTransactions()`/`GetOperations()`/`GetTrades()` for 6 other commands. This H003 covers the distinct `GetPaymentOperations()` code path used only by `export_assets`.

### Trace Summary

`GetPaymentOperations()` (assets.go:21) calls `utils.CreateLedgerBackend(ctx, false, env)` which builds a `BufferedStorageBackend` (main.go:1036) — no `ValidateLedgerRange()` call occurs anywhere in this chain. `BoundedRange(start, end)` (SDK range.go:77) simply constructs a struct with no validation. `PrepareRange()` (SDK buffered_storage_backend.go:172) stores the range and creates a buffer but does not check `from <= to`. The loop `for seq := start; seq <= end; seq++` (assets.go:31) silently produces zero iterations when `start > end`, returning an empty slice with nil error. Back in `export_assets` (export_assets.go:44-70), the empty `paymentOps` slice causes zero loop iterations, producing an empty JSON file that is closed and optionally uploaded to GCS via `MaybeUpload()`.

### Code Paths Examined

- `internal/input/assets.go:GetPaymentOperations:21-61` — calls `CreateLedgerBackend()`, then `PrepareRange(BoundedRange(start, end))`, then `for seq := start; seq <= end; seq++`; no `ValidateLedgerRange()` call; returns empty slice with nil error when loop is skipped
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:13-51` — the captive-core alternative; calls `utils.CreateBackend(start, end, ...)` which DOES call `ValidateLedgerRange()` at main.go:770, confirming the validation was expected for this kind of range read
- `internal/utils/main.go:CreateLedgerBackend:1011-1041` — datastore path builds `BufferedStorageBackend` with NO range validation; captive-core path calls `CreateCaptiveCoreBackend()` which also has NO range validation
- `internal/utils/main.go:CreateBackend:760-779` — the history-archive path calls `ValidateLedgerRange(start, end, root.CurrentLedger)` at line 770, confirming the pattern of validating ranges exists in the codebase
- `internal/utils/main.go:ValidateLedgerRange:736-758` — validates start≠0, end≠0, end≥start, and range within latest ledger; correctly rejects impossible ranges
- `cmd/export_assets.go:Run:17-83` — branches on `UseCaptiveCore`; the default (false) path calls `GetPaymentOperations()` without validation; accepts empty `paymentOps` as success; writes empty file and calls `MaybeUpload()`
- `internal/utils/main.go:MustArchiveFlags:541-562` — parses `start-ledger` (default 2) with no validation
- `internal/utils/main.go:MustCommonFlags:460-538` — parses `end-ledger` (default 0) with no validation
- `go-stellar-sdk/ingest/ledgerbackend/range.go:BoundedRange:77-79` — simply constructs `Range{from, to, true}` with no validation
- `go-stellar-sdk/ingest/ledgerbackend/buffered_storage_backend.go:PrepareRange:172-189` — stores range and creates buffer; does not validate `from <= to`

### Findings

The bug is confirmed for `export_assets` on the default (non-captive-core) path. Key observations:

1. **Validation gap is configuration-specific**: Unlike H002's commands where neither backend path validates, `export_assets` has a split: the captive-core path (`GetPaymentOperationsHistoryArchive` → `CreateBackend` → `ValidateLedgerRange`) DOES validate, but the default datastore path (`GetPaymentOperations` → `CreateLedgerBackend`) does NOT. This means the bug is present only on the default/production configuration.

2. **Distinct code path from H001/H002**: `GetPaymentOperations()` in `assets.go` is a separate function with its own loop and backend setup, not shared with any of the functions covered by H001 or H002. A fix to `GetTransactions()` or `GetLedgers()` would NOT fix this.

3. **Empty asset file uploads to GCS**: `export_assets` calls `MaybeUpload()` at line 77 after the export loop, so an empty file can be pushed to cloud storage. For asset exports specifically, this could cause downstream systems to conclude "no new assets were created in this range" when in fact the range was invalid.

4. **Same pattern, independent fix needed**: The fix requires adding `ValidateLedgerRange()` (or at minimum `if end < start || end == 0 { return error }`) at the top of `GetPaymentOperations()` or in the `export_assets` command before calling it. This is independent of fixes for H001/H002.

### PoC Guidance

- **Test file**: `internal/input/assets_test.go` or `cmd/export_assets_test.go` (whichever exists; otherwise create alongside existing test files in the same package)
- **Setup**: No live backend needed for demonstrating the validation gap. The PoC can call `GetPaymentOperations(100, 50, -1, env, false)` directly (which will fail at `CreateLedgerBackend` without GCS access), or more simply demonstrate the gap by:
  1. Showing `ValidateLedgerRange(100, 50, 1000)` returns an error
  2. Showing `GetPaymentOperations` does NOT call `ValidateLedgerRange` (trace the code path)
  3. Showing the loop `for seq := 100; seq <= 50; seq++` produces zero iterations
- **Steps**:
  1. Call `utils.ValidateLedgerRange(100, 50, 1000)` and verify it returns a non-nil error (proves the validator exists and works)
  2. Call `utils.ValidateLedgerRange(2, 0, 1000)` and verify it returns a non-nil error (covers end=0 case)
  3. Inspect `GetPaymentOperations` source to confirm no `ValidateLedgerRange` call exists (static analysis)
  4. Verify the loop condition `for seq := 100; seq <= 50; seq++` produces zero iterations (demonstrates empty result)
- **Assertion**: `ValidateLedgerRange` correctly rejects impossible ranges, but `GetPaymentOperations` never calls it. The fix would add a `ValidateLedgerRange()` call at the top of `GetPaymentOperations()` (requiring a `latestNum` lookup) or a simpler local guard `if end != 0 && end < start { return error }`.
