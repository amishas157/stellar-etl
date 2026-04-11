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
