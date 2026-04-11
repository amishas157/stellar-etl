# H002: Token-transfer export applies `--limit` to ledgers instead of emitted transfer rows

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Empty or truncated token-transfer exports under caller-visible row limits
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`stellar-etl export_token_transfer --limit N` should continue scanning the requested range until it has emitted `N` `TokenTransferOutput` rows or reached the end of the range. A ledger with zero token-transfer events should not consume the caller's token-transfer budget.

## Mechanism

The command passes `limit` into `input.GetLedgers()`, which stops after `N` ledgers. `transform.TransformTokenTransfer()` then expands each returned ledger into a variable-length slice of token-transfer rows based on the unified events processor, so the exported row count is determined by how many transfer events happen to exist inside the first `N` ledgers rather than by the caller's requested token-transfer count.

## Trigger

Run `stellar-etl export_token_transfer --limit 1 --start-ledger <S> --end-ledger <E>` on a range where ledger `S` contains no token-transfer events but ledger `S+1` contains at least one valid token-transfer event. The correct output is one transfer row from ledger `S+1`; the current code can return an empty file because ledger `S` consumed the entire limit before any row-level expansion happened.

## Target Code

- `cmd/export_token_transfers.go:21-29` — forwards the user-visible `token_transfer` limit into `GetLedgers()`
- `cmd/export_token_transfers.go:38-60` — exports every `TokenTransferOutput` row returned for each limited ledger
- `internal/input/ledgers.go:14-90` — enforces `limit` on `len(ledgerSlice)`, i.e. ledger count
- `internal/transform/token_transfer.go:14-34` — converts one ledger close meta into token-transfer events
- `internal/transform/token_transfer.go:37-129` — expands a single ledger's event list into a variable number of output rows
- `internal/utils/main.go:250-254` — defines `--limit` as the maximum number of `token_transfer` objects to export

## Evidence

`TransformTokenTransfer()` is explicitly fan-out logic: it calls `EventsFromLedger()`, verifies the event set, then appends one `TokenTransferOutput` per event. Because `GetLedgers()` knows nothing about token-transfer row counts and stops after a fixed number of ledgers, the first limited ledger can contribute zero rows while later ledgers with real transfers are skipped.

## Anti-Evidence

If each of the first `N` ledgers happens to emit exactly one token-transfer row, the observed row count matches the limit by accident. Leaving `--limit` negative also avoids the truncation because all ledgers in range are processed.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The `export_token_transfer` command at `cmd/export_token_transfers.go:28` passes the user-visible `limit` directly into `input.GetLedgers()`. Inside `GetLedgers()` (`internal/input/ledgers.go:85`), the limit check `int64(len(ledgerSlice)) >= limit` enforces the cap on the number of ledgers returned, not on the number of token-transfer rows. Back in the exporter loop (`cmd/export_token_transfers.go:38-54`), every `TokenTransferOutput` from `TransformTokenTransfer()` is exported without any secondary row cap. The flag help text from `AddArchiveFlags("token_transfer", ...)` at `internal/utils/main.go:254` explicitly says "Maximum number of token_transfer to export," contradicting the actual ledger-level enforcement.

### Code Paths Examined

- `cmd/export_token_transfers.go:21-28` — `limit` from `MustArchiveFlags` passed directly to `GetLedgers()`, no row-level tracking
- `internal/input/ledgers.go:24-88` — loop iterates `seq = start..end`, appends one `HistoryArchiveLedgerAndLCM` per ledger, breaks when `len(ledgerSlice) >= limit`
- `cmd/export_token_transfers.go:38-54` — iterates all returned ledgers, calls `TransformTokenTransfer()` on each, writes every resulting row; no counter or secondary limit
- `internal/transform/token_transfer.go:14-34` — `TransformTokenTransfer()` calls `EventsFromLedger()` which returns 0..N events per ledger, then `transformEvents()` produces one `TokenTransferOutput` per event
- `internal/utils/main.go:250-254` — `AddArchiveFlags("token_transfer", ...)` defines `--limit` as "Maximum number of token_transfer to export"
- `cmd/export_token_transfers.go:77` — inline comment contradicts flag text: "limit: maximum number of ledgers to export"

### Findings

The bug is confirmed: `--limit N` controls ledger count, not token-transfer row count. This is the same class of bug as the confirmed effects finding (`success/data-input/004`) and trades finding (`success/export-pipeline/010`), but applied to a distinct command and code path. The token transfer case is arguably the worst instance because the limit granularity mismatch spans two levels (ledger → transaction → token-transfer events) rather than one. A single ledger can contain dozens of transactions each producing multiple token-transfer events, making the overshoot potentially very large. Conversely, a ledger with zero token-transfer events silently consumes the limit budget and produces zero rows.

Severity downgraded from High to Medium for consistency with the effects precedent (same bug pattern, same operational-correctness impact). The issue breaks caller-visible batching semantics but does not corrupt monetary field values.

### PoC Guidance

- **Test file**: `internal/transform/token_transfer_test.go` (or a new `internal/transform/data_integrity_poc_test.go` if one exists)
- **Setup**: Build a `xdr.LedgerCloseMeta` containing one transaction with multiple token-transfer events (e.g., a payment + fee event). A second empty `LedgerCloseMeta` (no transactions). Call `TransformTokenTransfer()` on both.
- **Steps**: (1) Show that `TransformTokenTransfer()` on the empty ledger returns 0 rows. (2) Show that `TransformTokenTransfer()` on the event-bearing ledger returns >1 rows. (3) Demonstrate that `GetLedgers(start, end, limit=1, ...)` returns exactly 1 ledger regardless of token-transfer content, proving the limit is enforced at ledger granularity. (4) Show the flag help text via `AddArchiveFlags("token_transfer", ...)` says "token_transfer" not "ledgers."
- **Assertion**: A single ledger consumed by `--limit 1` can produce 0 rows (empty export) or N>1 rows (oversized export), violating the documented "Maximum number of token_transfer to export" semantics.
