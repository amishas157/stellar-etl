# H010: Ignored `LedgerTransactionReader.Read()` errors silently corrupt transaction-based exports

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: Medium
**Impact**: Suspected silent corruption from dropped transaction-reader errors
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`GetTransactions()`, `GetOperations()`, and `GetTrades()` should propagate any non-EOF `txReader.Read()` error instead of continuing, because a legitimate read failure should stop the export rather than quietly producing incomplete or zero-valued rows.

## Mechanism

Each reader only checks `err == io.EOF` after `txReader.Read()` and otherwise proceeds as if the returned transaction were valid. If `Read()` could fail on legitimate ledger data, the readers might append incomplete transactions or silently truncate later rows without surfacing the underlying problem.

## Trigger

Process a legitimate Stellar ledger for which `ingest.LedgerTransactionReader.Read()` returns a non-EOF error during normal iteration.

## Target Code

- `internal/input/transactions.go:50-62` — ignores non-EOF `txReader.Read()` errors
- `internal/input/operations.go:52-70` — same omission in the operations reader
- `internal/input/trades.go:50-76` — same omission in the trades reader
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction_reader.go:76-105` — `Read()` implementation and its actual non-EOF error branch

## Evidence

The in-repo readers undeniably discard any non-EOF read error, which is usually dangerous in a streaming loop. The SDK's `Read()` method does have a non-EOF error branch (`unknown tx hash in LedgerCloseMeta`), so the omission is not purely theoretical at the API level.

## Anti-Evidence

Tracing the SDK shows that the non-EOF branch requires malformed ledger-close-meta contents: `Read()` only errors when the close meta references a transaction hash that was not present in the stored envelopes, while `NewLedgerTransactionReaderFromLedgerCloseMeta()` already validates the normal per-ledger setup up front. That makes the remaining read-time failure mode a corrupt backend/SDK mismatch issue rather than a bug triggered by legitimate on-chain data.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

I could not identify a valid Stellar ledger that makes `Read()` return a non-EOF error during normal export. The surviving error branch depends on malformed `LedgerCloseMeta` contents, which falls outside this objective's "legitimate chain data produces plausible-but-wrong output" scope.

### Lesson Learned

When a reader ignores an error, inspect the upstream implementation before escalating it. An omitted error check is only a data-integrity finding if the ignored condition is reachable from real chain data, not just from corrupted backend state.
