# H004: `export_token_transfer --limit` counts ledgers, not emitted transfer rows

**Date**: 2026-04-13
**Subsystem**: external-io
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

The shared archive flag text says `--limit` is the maximum number of exported `token_transfer` rows, so `export_token_transfer --limit N` should emit at most `N` token-transfer records, and `--limit 0` should emit none.

## Mechanism

`export_token_transfer` passes `limit` into `input.GetLedgers()`, which counts ledgers rather than transfer rows and only checks the limit after appending a ledger to `ledgerSlice`. The command then calls `TransformTokenTransfer()` once per selected ledger, and that transform emits one row per token-transfer event in the ledger. A single included ledger can therefore exceed the requested row cap, and `limit=0` still processes the first ledger in range.

## Trigger

1. Run `stellar-etl export_token_transfer --limit 0` over any ledger range whose first ledger contains at least one token-transfer event.
2. Or run `stellar-etl export_token_transfer --limit 1` over a range whose first included ledger contains multiple token-transfer events.

## Target Code

- `internal/utils/main.go:250-254` — advertises `--limit` as the maximum number of `token_transfer` objects to export
- `cmd/export_token_transfers.go:21-28` — passes the user limit into `GetLedgers()`
- `internal/input/ledgers.go:21-27,84-86` — appends a full ledger before enforcing `limit`
- `cmd/export_token_transfers.go:38-54` — writes every token-transfer row returned from each selected ledger
- `internal/transform/token_transfer.go:14-35` — derives token-transfer events from one ledger close meta
- `internal/transform/token_transfer.go:37-129` — appends one output row per token-transfer event in that ledger

## Evidence

The limit boundary is at the ledger-reader layer, not the emitted-row layer. `GetLedgers()` can return one full ledger even when `limit=0`, and `TransformTokenTransfer()` is explicitly row-expanding because it loops over every event returned by `EventsFromLedger()`.

## Anti-Evidence

Ledgers with zero or one token-transfer event will not visibly exceed a positive limit. The individual exported rows can still be internally correct; the bug is that the CLI silently violates the requested output bound.

---

## Review

**Verdict**: NOT_VIABLE — duplicate of success/data-input/006-token-transfer-limit-counts-ledgers-instead-of-token-transfer-rows.md.gh-published
**Date**: 2026-04-13
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of success/data-input/006-token-transfer-limit-counts-ledgers-instead-of-token-transfer-rows.md.gh-published
**Failed At**: reviewer

### Trace Summary

This hypothesis describes the exact same bug already confirmed and published as `success/data-input/006-token-transfer-limit-counts-ledgers-instead-of-token-transfer-rows.md.gh-published`. That success file documents the same mechanism (`GetLedgers()` enforces limit on ledger count, `TransformTokenTransfer()` expands each ledger into multiple rows), the same trigger (`--limit 1` on a multi-event ledger), and includes real pubnet reproductions. Additionally, the identical hypothesis was previously filed under `cli-commands` and rejected in `fail/cli-commands/026-token-transfer-limit-counts-ledgers.md` as a duplicate of the same success entry.

### Code Paths Examined

- `cmd/export_token_transfers.go:28` — passes limit to `GetLedgers()` (confirmed, same as prior investigation)
- `internal/input/ledgers.go:84-86` — limit enforced at ledger granularity (confirmed, same as prior investigation)

### Why It Failed

Exact duplicate of an already-confirmed finding. The token-transfer limit-counts-ledgers bug has been investigated, confirmed with PoC, and published under `data-input/006`. A second duplicate was already rejected under `cli-commands/026`.

### Lesson Learned

The `--limit` counting-at-wrong-granularity pattern for multi-row-expansion commands has been thoroughly investigated across multiple subsystems. Check `success/data-input/` for prior confirmed findings on this pattern before filing under a different subsystem.
