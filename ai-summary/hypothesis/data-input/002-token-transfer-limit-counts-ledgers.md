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
