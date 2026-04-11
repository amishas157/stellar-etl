# H003: Continuous Ledger-Change Export Prepares a Bounded `[start,0]` Range

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: Medium
**Impact**: Operational correctness: the documented continuous export mode does not stream any live batches
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes` is run without `--end-ledger`, it should prepare an unbounded backend range and continue exporting newly closed ledgers as the command help promises. The first batch should begin at `start-ledger`, and later ledgers should keep arriving as the network advances.

## Mechanism

The command calls `backend.PrepareRange(ctx, ledgerbackend.BoundedRange(startNum, commonArgs.EndNum))` while `commonArgs.EndNum` is still `0`, and only afterwards rewrites `EndNum` to `math.MaxInt32` before starting `StreamChanges()`. In the SDK, `BoundedRange(from, to)` always produces a bounded range even when `to == 0`; it is `UnboundedRange(from)` that enables streaming mode. That leaves the backend prepared for a nonsensical bounded `[start,0]` window, so subsequent reads of `start` are outside the prepared range and the advertised continuous-export path never produces live change batches.

## Trigger

1. Run `stellar-etl export_ledger_entry_changes --start-ledger <L>` without `--end-ledger`.
2. Expected behavior: the process stays alive and begins exporting batch files as new ledgers arrive.
3. Actual behavior: the backend is prepared against `[L,0]` before the command switches to its pseudo-unbounded `math.MaxInt32` loop, so the first live read fails instead of streaming batches.

## Target Code

- `cmd/export_ledger_entry_changes.go:26-27` — promises continuous export when `--end-ledger` is omitted
- `cmd/export_ledger_entry_changes.go:61-78` — prepares a bounded range with `EndNum == 0`, then only afterwards rewrites `EndNum`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledgerbackend/range.go:76-83` — `BoundedRange()` remains bounded; `UnboundedRange()` is the only streaming constructor
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledgerbackend/ledger_buffer.go:98-104` — bounded mode stops queueing tasks past the end boundary ledger
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledgerbackend/buffered_storage_backend.go:138-145` — reads beyond the prepared bounded range return `requested sequence beyond current LedgerRange`

## Evidence

The rewrite to `math.MaxInt32` happens after `PrepareRange()` has already committed the backend to a bounded range. The SDK code makes the distinction explicit: `BoundedRange(start, 0)` is not a special case for "stream forever", and the buffered backend later rejects any request whose sequence exceeds the prepared `to` boundary of `0`.

## Anti-Evidence

Bounded exports with an explicit `--end-ledger` do not hit this path. This is limited to the command's documented continuous mode, but that mode is a first-class feature described in the command help rather than an internal helper.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The command at `cmd/export_ledger_entry_changes.go:67` calls `backend.PrepareRange(ctx, ledgerbackend.BoundedRange(startNum, commonArgs.EndNum))` when `EndNum == 0` (no `--end-ledger` flag). This creates `Range{from: startNum, to: 0, bounded: true}` — a backwards bounded range. Only after this call (line 72-74) does the code rewrite `EndNum` to `math.MaxInt32`. The correct pattern already exists in the same codebase at `internal/input/changes.go:59-62` (`PrepareCaptiveCore`) which defaults to `UnboundedRange(start)` and only switches to `BoundedRange` when `end != 0`.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:67` — `PrepareRange(ctx, BoundedRange(startNum, 0))` called before EndNum rewrite
- `cmd/export_ledger_entry_changes.go:72-74` — EndNum rewritten to `math.MaxInt32` AFTER PrepareRange already committed the backend
- `internal/input/changes.go:59-62` — `PrepareCaptiveCore` correctly uses `UnboundedRange(start)` as default, only switching to `BoundedRange(start, end)` when `end != 0`
- SDK `range.go:77-78` — `BoundedRange(from, to)` always sets `bounded: true`; no special-case for `to == 0`
- SDK `captive_core_backend.go:472-476` — CaptiveCore dispatches on `ledgerRange.bounded`: bounded → `openOfflineReplaySubprocess(startNum, 0)` (invalid catchup range), unbounded → `openOnlineReplaySubprocess` (streaming mode)
- SDK `buffered_storage_backend.go:142-145` — BufferedStorageBackend rejects `GetLedger(startNum)` when `startNum > to` (which is 0), returning "requested sequence beyond current LedgerRange"
- `internal/input/changes.go:111-113` — `extractBatch` calls `ingest.NewLedgerChangeReader` which calls `GetLedger`, propagating the error to `logger.Fatal`

### Findings

The bug is confirmed. Two backend types are affected:

1. **CaptiveCore**: `startPreparingRange` checks `ledgerRange.bounded == true` and calls `openOfflineReplaySubprocess(startNum, 0)`. This starts stellar-core in offline catchup mode with an invalid backwards range (`from > to`), which will fail during `PrepareRange` itself (line 68-69 in the command, `cmdLogger.Fatal`).

2. **BufferedStorageBackend**: `PrepareRange` succeeds (just stores the range), but the first `GetLedger(startNum)` call fails because `startNum > 0 == to`, producing "requested sequence beyond current LedgerRange". This triggers `logger.Fatal` in `extractBatch`.

In both cases the failure is **visible** (Fatal crash), not silent data corruption. The continuous export mode is a documented first-class feature (command help, line 26-27) that simply does not work. The fix is straightforward: use `UnboundedRange(startNum)` when `EndNum == 0`, matching the pattern already present in `PrepareCaptiveCore`.

### PoC Guidance

- **Test file**: `internal/input/changes_test.go` (or a new `cmd/export_ledger_entry_changes_test.go`)
- **Setup**: Create a mock `LedgerBackend` that records the `Range` passed to `PrepareRange`. No real network or stellar-core needed.
- **Steps**: Simulate the command's logic: call `PrepareRange(ctx, BoundedRange(startNum, 0))` where `startNum > 0`, then attempt `GetLedger(startNum)`.
- **Assertion**: Assert that `PrepareRange` receives `BoundedRange(startNum, 0)` with `bounded == true` and `to == 0`, confirming the backend is prepared with a nonsensical range. Alternatively, assert that `GetLedger(startNum)` returns an error when the prepared range has `to == 0`. Compare against the correct behavior: `PrepareRange(ctx, UnboundedRange(startNum))` should succeed.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/input/data_integrity_poc_test.go
**Test Name**: "TestContinuousModePreparesNonsensicalBoundedRange"
**Test Language**: Go

### Demonstration

The test replicates the exact range construction logic from `cmd/export_ledger_entry_changes.go:67` (which calls `BoundedRange(startNum, 0)` when EndNum is 0) and compares it against the correct pattern from `internal/input/changes.go:59-62` (which uses `UnboundedRange(startNum)` when end is 0). The test proves the command produces a bounded range `[100,0]` with `from > to`, a nonsensical backwards range that any backend will reject, while the correct code produces `[100,latest)` — an unbounded streaming range.

### Test Body

```go
package input

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest/ledgerbackend"
)

// TestContinuousModePreparesNonsensicalBoundedRange demonstrates that the
// export_ledger_entry_changes command constructs a BoundedRange(startNum, 0)
// when --end-ledger is omitted, instead of using UnboundedRange(startNum).
// This creates a nonsensical backwards bounded range [start, 0] that the
// backend will reject.
func TestContinuousModePreparesNonsensicalBoundedRange(t *testing.T) {
	startNum := uint32(100)
	endNum := uint32(0) // simulates --end-ledger not provided (default)

	// ---- What the command does (cmd/export_ledger_entry_changes.go:67) ----
	// backend.PrepareRange(ctx, ledgerbackend.BoundedRange(startNum, commonArgs.EndNum))
	// At this point EndNum is still 0 because the rewrite to math.MaxInt32
	// happens AFTER PrepareRange (lines 72-74).
	cmdRange := ledgerbackend.BoundedRange(startNum, endNum)

	// ---- What PrepareCaptiveCore correctly does (internal/input/changes.go:59-62) ----
	var correctRange ledgerbackend.Range
	if endNum != 0 {
		correctRange = ledgerbackend.BoundedRange(startNum, endNum)
	} else {
		correctRange = ledgerbackend.UnboundedRange(startNum)
	}

	// Verify the command produces a nonsensical bounded range [100, 0]
	if !cmdRange.Bounded() {
		t.Fatal("expected command range to be bounded (reproducing the bug path)")
	}
	if cmdRange.To() != 0 {
		t.Fatalf("expected command range To() == 0, got %d", cmdRange.To())
	}
	if cmdRange.From() <= cmdRange.To() {
		t.Fatalf("expected From() > To() (nonsensical backwards range), got From=%d To=%d",
			cmdRange.From(), cmdRange.To())
	}

	// Verify the correct implementation produces an unbounded streaming range
	if correctRange.Bounded() {
		t.Fatal("expected correct range to be unbounded for continuous mode")
	}
	if correctRange.From() != startNum {
		t.Fatalf("expected correct range From() == %d, got %d", startNum, correctRange.From())
	}

	// The bug: the command's range is bounded with to=0, making it impossible
	// to read any ledger >= startNum. The String() output makes this visible.
	cmdStr := cmdRange.String()
	correctStr := correctRange.String()
	t.Logf("Command produces range: %s (bounded=%v, from=%d, to=%d)",
		cmdStr, cmdRange.Bounded(), cmdRange.From(), cmdRange.To())
	t.Logf("Correct range should be: %s (bounded=%v, from=%d)",
		correctStr, correctRange.Bounded(), correctRange.From())

	// The command range is [100,0] — backwards and will be rejected by the backend.
	// The correct range is [100,latest) — unbounded streaming mode.
	if cmdRange.Bounded() == correctRange.Bounded() {
		t.Fatal("expected command range and correct range to differ in boundedness — bug not reproduced")
	}

	t.Log("BUG CONFIRMED: export_ledger_entry_changes continuous mode calls " +
		"PrepareRange with BoundedRange(start, 0) instead of UnboundedRange(start)")
}
```

### Test Output

```
=== RUN   TestContinuousModePreparesNonsensicalBoundedRange
    data_integrity_poc_test.go:56: Command produces range: [100,0] (bounded=true, from=100, to=0)
    data_integrity_poc_test.go:58: Correct range should be: [100,latest) (bounded=false, from=100)
    data_integrity_poc_test.go:67: BUG CONFIRMED: export_ledger_entry_changes continuous mode calls PrepareRange with BoundedRange(start, 0) instead of UnboundedRange(start)
--- PASS: TestContinuousModePreparesNonsensicalBoundedRange (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/input	0.820s
```

---

## Final Review

**Verdict**: REJECTED
**Date**: 2026-04-11
**Final review by**: gpt-5.4, high
**Failed At**: final-review

### Adversarial Analysis

1. **Does the PoC actually exercise the claimed issue?** Yes after revision. I rewrote the PoC to use the real `ingest.NewLedgerChangeReader()` path with a real `ledgerbackend.BufferedStorageBackend`. Preparing `BoundedRange(start, 0)` leaves the backend prepared as bounded, and the first read for `start` fails with `requested sequence beyond current LedgerRange`.
2. **Are the preconditions realistic?** Partially. The CLI does execute `PrepareRange(ctx, ledgerbackend.BoundedRange(startNum, commonArgs.EndNum))` before rewriting `EndNum`, so the bad range is real. But this only matters in the command's unbounded mode.
3. **Is the behavior a bug or by design?** By design for the current product surface. The repository README explicitly documents `export_ledger_entry_changes` unbounded mode as `Currently Unsupported` (`README.md:295-303`). That means the PoC demonstrates failure in an unsupported mode, not a confirmed supported-feature defect.
4. **Does the impact match the claimed severity?** No. The observed behavior is an immediate, visible failure to start streaming, not silent data corruption or masked data loss. As framed, it does not satisfy the data-integrity objective for this review.
5. **Is the finding in scope?** No as written. The issue is a mismatch between command help text and the repository's command documentation for an unsupported mode, not an in-scope data correctness bug producing wrong exported data.
6. **Is the test itself correct?** The revised test was correct. It proved the prepared range remains bounded `[start,0]` and that the first ledger read fails for that reason.
7. **Can the results be explained without the claimed issue?** No. The bad prepared range is real. The disqualifier is scope/support status, not reproduction.
8. **Is this finding novel?** Possibly, but novelty does not overcome the unsupported-mode/by-design problem.

### Rejection Reason

The PoC demonstrates a real failure path, but the command's own README documents unbounded ledger-entry-change export as **currently unsupported**. That makes this an unsupported-mode/help-text inconsistency rather than a confirmed, in-scope data-integrity finding.

### Failed Checks

- 3
- 4
- 5
