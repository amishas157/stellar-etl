# H001: `export_ledger_entry_changes --batch-size 1` drops valid multi-ledger exports

**Date**: 2026-04-12
**Subsystem**: cli-commands
**Severity**: High
**Impact**: empty results from broken control flow
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes` is run with `--batch-size 1`, every valid ledger in the requested inclusive range should produce its own `ChangeBatch`. For a three-ledger range `[L, L+2]`, the command should therefore emit three batch file sets, each covering exactly one ledger.

## Mechanism

`StreamChanges()` initializes `batchEnd` as `min(batchStart+batchSize, end)` and then decrements it when `batchEnd < end`, but it still uses the strict guard `for batchStart < batchEnd`. With `batchSize == 1` and any range longer than two ledgers, that decrement makes the first `batchEnd` equal `batchStart`, so the loop never executes and no batch is ever sent. `export_ledger_entry_changes` then observes a clean channel shutdown and exits successfully with an empty export even though the requested ledgers are valid and may contain changes.

## Trigger

Run:

`stellar-etl export_ledger_entry_changes -x ../stellar-core/src/stellar-core -c /etl/docker/stellar-core.cfg -s 49265302 -e 49265304 -b 1 -o /tmp/changes`

The current implementation should export three one-ledger batches, but instead it can finish without writing any batch files at all.

## Target Code

- `internal/input/changes.go:StreamChanges:162-177` — decrements `batchEnd` and then skips the loop entirely when `batchStart == batchEnd`
- `cmd/export_ledger_entry_changes.go:Run:76-86` — treats the closed `changeChan` / `closeChan` sequence as normal completion when no batch was ever emitted
- `internal/utils/main.go:AddCoreFlags:271-277` — documents `batch-size` as the number of ledgers per batch
- `internal/input/changes_test.go:TestStreamChangesBatchNumbers:154-229` — only exercises the default `batchSize=64` path, so the `batch-size 1` failure is currently uncovered

## Evidence

The batching math is currently written around inclusive `ExtractBatch(batchStart, batchEnd)` calls, but the loop guard is still strict `<`. For `start=49265302`, `end=49265304`, `batchSize=1`, the first computed `batchEnd` is `49265303`, the `if batchEnd < end` branch decrements it to `49265302`, and the loop condition `49265302 < 49265302` immediately fails. The command-side consumer has no independent check that at least one batch was produced.

## Anti-Evidence

There is already a published one-ledger batching bug for the equality case `start == end`, so reviewers may suspect this is just the same issue restated. The distinction is that this trigger does not rely on a single-ledger range at all: it uses a valid multi-ledger range and a documented small `batch-size`, and it still collapses to a success-shaped empty export.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced `StreamChanges` (changes.go:162-178) step-by-step with `start=49265302, end=49265304, batchSize=1`. The hypothesis claims the loop never executes because the decrement at line 167 makes `batchEnd == batchStart` before the loop guard is checked. This is factually wrong: the loop guard at line 165 (`batchStart < batchEnd`) is evaluated BEFORE the body executes, using the pre-decrement value. The initial `batchEnd = min(start+1, end) = 49265303`, so `49265302 < 49265303` is true and the loop enters. The decrement happens inside the loop body on the first iteration, not before the first check.

### Code Paths Examined

- `internal/input/changes.go:StreamChanges:162-178` — Loop guard at line 165 uses the un-decremented batchEnd (49265303), so `49265302 < 49265303` passes. The decrement at line 167 adjusts batchEnd to 49265302 INSIDE the loop body, after entry.
- `internal/input/changes.go:extractBatch:82-158` — Compactors are created per-ledger (line 102-105 inside the `for seq` loop), so multiple ledgers within one batch are NOT cross-compacted. Each ledger's changes are independently compacted then appended.
- `cmd/export_ledger_entry_changes.go:76-86` — Consumer loop receives batches; irrelevant since batches ARE produced.

### Why It Failed

The hypothesis misreads the control flow. It claims the `batchEnd` decrement (line 167) happens before the loop guard (line 165), but in Go's `for` loop, the condition is evaluated first. With `batchSize=1` on range [49265302, 49265304]:

1. Init: `batchEnd = min(49265303, 49265304) = 49265303`
2. Loop check: `49265302 < 49265303` → TRUE (enters loop)
3. Decrement inside body: `batchEnd = 49265302`
4. `ExtractBatch(49265302, 49265302)` — 1 ledger ✓
5. Advance: `batchStart=49265303, batchEnd=49265304`
6. Loop check: `49265303 < 49265304` → TRUE (enters again)
7. No decrement (49265304 is not < 49265304)
8. `ExtractBatch(49265303, 49265304)` — 2 ledgers

The loop produces 2 batches containing all 3 ledgers. No data is dropped. The claim of "empty results" and "no batch is ever sent" is factually incorrect.

Note: The last batch IS oversized (2 ledgers instead of 1), but this is a batch-boundary cosmetic issue, not a data correctness bug: `extractBatch` creates fresh compactors per ledger (line 102), so per-ledger change compaction is identical regardless of batch boundaries.

### Lesson Learned

When a hypothesis claims a `for` loop never executes due to an in-body mutation, verify the evaluation order. Go evaluates the loop condition BEFORE the body on every iteration including the first. A decrement inside the body cannot prevent the first entry if the initial condition is satisfied. Always trace the actual init → check → body → advance sequence step by step.
