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
