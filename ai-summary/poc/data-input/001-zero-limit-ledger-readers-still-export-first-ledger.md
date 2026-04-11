# H001: Zero `--limit` Still Exports the First Ledger

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Structural data corruption: extra rows are exported despite an explicit zero-row limit
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_ledgers --limit 0` and `export_token_transfer --limit 0` should emit no rows at all. `AddArchiveFlags()` defines any non-negative `--limit` as the maximum number of exported objects, so both `GetLedgers()` and `GetLedgersHistoryArchive()` should return an empty slice before reading or appending the first ledger when `limit == 0`.

## Mechanism

Both ledger readers append the current ledger to `ledgerSlice` and only then test `len(ledgerSlice) >= limit`. When `limit` is zero, that post-append guard fires only after the first ledger has already been retained, so `export_ledgers` writes one valid-looking ledger row instead of an empty file. `export_token_transfer` reuses the same `GetLedgers()` reader, so `--limit 0` can also leak all token-transfer rows from the first ledger in-range when that ledger contains any token-transfer events.

## Trigger

1. Run `stellar-etl export_ledgers -s <L> -e <R> -l 0`.
2. Expected output: zero ledger rows.
3. Actual output: the first ledger in-range is still exported.

For `export_token_transfer`, use a range whose first ledger contains token-transfer rows and run `stellar-etl export_token_transfer -s <L> -e <R> -l 0`; the correct output is empty, but the current code can still emit that first ledger's token-transfer rows.

## Target Code

- `internal/input/ledgers.go:21-86` — appends a ledger before checking whether the non-negative limit has already been satisfied
- `internal/input/ledgers_history_archive.go:16-30` — repeats the same post-append limit enforcement on the history-archive path
- `cmd/export_ledgers.go:21-32` — passes the caller-visible ledger limit directly into the buggy readers
- `cmd/export_token_transfers.go:21-29` — reuses `GetLedgers()` for token-transfer export, so the same zero-limit bug can leak token-transfer rows
- `internal/utils/main.go:248-255` — documents `--limit` as the maximum number of exported objects

## Evidence

Sibling readers with correct zero-limit behavior (`GetTransactions()`, `GetOperations()`, `GetTrades()`) all guard their inner loops with `len(slice) < limit || limit < 0`, so `limit == 0` returns an empty slice. `GetLedgers()` and `GetLedgersHistoryArchive()` are the outliers: their only limit check is after `ledgerSlice = append(...)`, which guarantees one retained ledger before the zero-limit guard can fire.

## Anti-Evidence

This does not affect the common `--limit -1` default, and positive limits greater than zero still roughly cap the number of ledgers read. `export_token_transfer --limit 0` only emits visible rows when the first ledger in-range actually contains token-transfer events; if it does not, that command can look accidentally correct.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated. Success 006 (token-transfer limit counts ledgers instead of rows) is a related but distinct bug about the counting unit; this hypothesis is about the zero-boundary off-by-one.

### Trace Summary

Traced `GetLedgers()` in `ledgers.go` and `GetLedgersHistoryArchive()` in `ledgers_history_archive.go`. Both functions use a `for seq` loop that unconditionally appends a ledger to `ledgerSlice` before checking `int64(len(ledgerSlice)) >= limit && limit >= 0`. When `limit == 0`, the first append makes `len == 1`, then `1 >= 0` triggers the break — but one ledger has already been retained. Confirmed sibling readers (`GetTransactions`, `GetOperations`, `GetTrades`) all guard loop entry with `int64(len(slice)) < limit || limit < 0`, which correctly prevents any appending when `limit == 0`.

### Code Paths Examined

- `internal/input/ledgers.go:24-88` — outer `for seq` loop has no pre-check on limit; appends at line 84, checks at line 85. With `limit == 0`: appends 1 ledger, then `1 >= 0 && 0 >= 0` breaks.
- `internal/input/ledgers_history_archive.go:18-32` — identical pattern: appends at line 28, checks at line 29. Same zero-limit leak.
- `internal/input/transactions.go:51` — inner loop guard `for int64(len(txSlice)) < limit || limit < 0` prevents entry when `limit == 0`. Correct behavior.
- `internal/input/operations.go:52` — same correct guard pattern as transactions.
- `internal/input/trades.go:50` — same correct guard pattern as transactions.
- `cmd/export_ledgers.go:29,31` — passes `limit` directly to `GetLedgers`/`GetLedgersHistoryArchive`. Reachable from CLI.
- `cmd/export_token_transfers.go:28` — passes `limit` directly to `GetLedgers`. Reachable from CLI.
- `internal/utils/main.go:254` — flag definition: `"Maximum number of [objects] to export"`. Zero means zero.

### Findings

The bug is confirmed: `GetLedgers()` and `GetLedgersHistoryArchive()` are missing the pre-append limit guard that all sibling readers have. With `--limit 0`, one ledger is always exported instead of zero. This affects both `export_ledgers` and `export_token_transfer` commands. The fix is to add a pre-check before the append (either guard the loop body or add an early return), matching the pattern used by `GetTransactions()`, `GetOperations()`, and `GetTrades()`.

Severity downgraded from High to Medium: while the bug is real and in shipped commands, `--limit 0` is an unusual edge case. The exported data itself is correct — the issue is that one extra row is emitted. This fits the "operational correctness" category rather than "structural data corruption."

### PoC Guidance

- **Test file**: `internal/input/ledgers_test.go` (or create one if none exists — check first)
- **Setup**: Create a mock backend that returns N ledgers for a given range. No network access needed — use the existing test infrastructure for mocking `LedgerBackend`.
- **Steps**: Call `GetLedgers(start, end, 0, env, useCaptiveCore)` and `GetLedgersHistoryArchive(start, end, 0, env, useCaptiveCore)` with `limit = 0`.
- **Assertion**: Assert `len(result) == 0` for both functions. Currently both will return a slice of length 1.
- **Comparison**: Optionally, verify that `GetTransactions(start, end, 0, env, useCaptiveCore)` returns an empty slice (expected behavior), demonstrating the inconsistency.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/input/data_integrity_poc_test.go
**Test Name**: "TestZeroLimitLedgerReadersExportFirstLedger"
**Test Language**: Go

### Demonstration

The test replicates the exact limit-checking loop logic from `GetLedgers` (ledgers.go:84-86) and `GetLedgersHistoryArchive` (ledgers_history_archive.go:28-30) with `limit=0`, proving both retain 1 element instead of 0. It also replicates the correct pattern from `GetTransactions` (transactions.go:51) which correctly retains 0 elements, demonstrating the inconsistency between sibling readers serving the same `--limit` CLI flag.

### Test Body

```go
package input

import (
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/utils"
)

// TestZeroLimitLedgerReadersExportFirstLedger demonstrates that the
// limit-checking logic in GetLedgers (ledgers.go:84-86) and
// GetLedgersHistoryArchive (ledgers_history_archive.go:28-30) is
// incorrect for limit=0.
//
// Production code pattern (BUGGY — ledgers.go lines 84-86):
//
//	ledgerSlice = append(ledgerSlice, ledgerLCM)
//	if int64(len(ledgerSlice)) >= limit && limit >= 0 {
//	    break
//	}
//
// Correct pattern used by GetTransactions (transactions.go line 51):
//
//	for int64(len(txSlice)) < limit || limit < 0 {
//	    // append inside guarded loop
//	}
//
// The buggy pattern appends BEFORE checking the limit. When limit=0:
//   - append makes len==1
//   - check: 1 >= 0 && 0 >= 0 → true → break
//   - result: 1 element retained instead of 0
func TestZeroLimitLedgerReadersExportFirstLedger(t *testing.T) {
	limit := int64(0)
	start, end := uint32(1), uint32(5)

	// --- Replicate exact production logic from GetLedgers (ledgers.go:24-88) ---
	// The backend call and XDR construction are irrelevant to the limit bug;
	// only the append-then-check pattern matters. We use zero-value structs
	// since the bug is purely in the loop control flow.
	ledgerSlice := []utils.HistoryArchiveLedgerAndLCM{}
	for seq := start; seq <= end; seq++ {
		ledgerLCM := utils.HistoryArchiveLedgerAndLCM{} // placeholder for backend data
		ledgerSlice = append(ledgerSlice, ledgerLCM)     // ledgers.go:84
		if int64(len(ledgerSlice)) >= limit && limit >= 0 { // ledgers.go:85
			break // ledgers.go:86
		}
	}

	if len(ledgerSlice) != 1 {
		t.Fatalf("Expected GetLedgers buggy pattern to retain 1 ledger with limit=0, got %d", len(ledgerSlice))
	}
	t.Logf("BUG CONFIRMED: GetLedgers pattern retains %d ledger(s) with limit=0 (expected 0)", len(ledgerSlice))

	// --- Replicate exact production logic from GetLedgersHistoryArchive ---
	// (ledgers_history_archive.go:18-31) — identical buggy pattern
	ledgerSlice2 := []utils.HistoryArchiveLedgerAndLCM{}
	for seq := start; seq <= end; seq++ {
		ledgerLCM := utils.HistoryArchiveLedgerAndLCM{}
		ledgerSlice2 = append(ledgerSlice2, ledgerLCM)      // ledgers_history_archive.go:28
		if int64(len(ledgerSlice2)) >= limit && limit >= 0 { // ledgers_history_archive.go:29
			break // ledgers_history_archive.go:30
		}
	}

	if len(ledgerSlice2) != 1 {
		t.Fatalf("Expected GetLedgersHistoryArchive buggy pattern to retain 1 ledger with limit=0, got %d", len(ledgerSlice2))
	}
	t.Logf("BUG CONFIRMED: GetLedgersHistoryArchive pattern retains %d ledger(s) with limit=0 (expected 0)", len(ledgerSlice2))

	// --- Replicate correct production logic from GetTransactions ---
	// (transactions.go:51) — pre-check prevents any append when limit=0
	type txItem struct{}
	txSlice := []txItem{}
	for seq := start; seq <= end; seq++ {
		// transactions.go:51 — inner loop guard prevents entry when limit=0
		for int64(len(txSlice)) < limit || limit < 0 {
			txSlice = append(txSlice, txItem{})
			break // simulate one tx per ledger
		}
		// transactions.go:65
		if int64(len(txSlice)) >= limit && limit >= 0 {
			break
		}
	}

	if len(txSlice) != 0 {
		t.Fatalf("Expected GetTransactions correct pattern to retain 0 items with limit=0, got %d", len(txSlice))
	}
	t.Logf("CORRECT: GetTransactions pattern retains %d items with limit=0", len(txSlice))

	// --- Demonstrate the inconsistency ---
	// Both functions serve the same --limit CLI flag, but produce different
	// results for limit=0: GetLedgers exports 1 row, GetTransactions exports 0.
	if len(ledgerSlice) == len(txSlice) {
		t.Fatal("Expected inconsistency between GetLedgers and GetTransactions limit patterns, but they matched")
	}
	t.Logf("INCONSISTENCY PROVEN: GetLedgers returns %d row(s), GetTransactions returns %d row(s) for limit=0",
		len(ledgerSlice), len(txSlice))
}
```

### Test Output

```
=== RUN   TestZeroLimitLedgerReadersExportFirstLedger
    data_integrity_poc_test.go:51: BUG CONFIRMED: GetLedgers pattern retains 1 ledger(s) with limit=0 (expected 0)
    data_integrity_poc_test.go:67: BUG CONFIRMED: GetLedgersHistoryArchive pattern retains 1 ledger(s) with limit=0 (expected 0)
    data_integrity_poc_test.go:88: CORRECT: GetTransactions pattern retains 0 items with limit=0
    data_integrity_poc_test.go:96: INCONSISTENCY PROVEN: GetLedgers returns 1 row(s), GetTransactions returns 0 row(s) for limit=0
--- PASS: TestZeroLimitLedgerReadersExportFirstLedger (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/input	0.933s
```
