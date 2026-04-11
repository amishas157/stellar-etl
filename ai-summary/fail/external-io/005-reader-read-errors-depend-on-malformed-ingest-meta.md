# H005: Input readers swallow `txReader.Read()` errors, but the trigger depends on malformed ingest metadata

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `txReader.Read()` returns a non-EOF error while reading a ledger, the input helpers should surface that error to the caller instead of continuing with partial results. Export commands should fail the affected ledger range rather than silently skipping transactions, operations, trades, or derived rows.

## Mechanism

`GetTransactions()`, `GetOperations()`, and `GetTrades()` only special-case `io.EOF` and otherwise continue past `txReader.Read()` errors. Because `LedgerTransactionReader.Read()` advances `readIdx` before returning an error, a non-EOF error would drop the bad transaction and let the loop continue with later ones. I investigated this as a possible silent-omission bug.

## Trigger

Force `ingest.LedgerTransactionReader.Read()` to hit a non-EOF error such as `unknown tx hash in LedgerCloseMeta` or `could not hash transaction ...`, then call `GetTransactions()`, `GetOperations()`, or `GetTrades()` on that ledger range.

## Target Code

- `internal/input/transactions.go:GetTransactions:51-61` - ignores non-EOF `txReader.Read()` errors and still appends the returned transaction value
- `internal/input/operations.go:GetOperations:52-71` - ignores non-EOF `txReader.Read()` errors before expanding operations
- `internal/input/trades.go:GetTrades:50-77` - ignores non-EOF `txReader.Read()` errors before collecting trade-producing operations
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction_reader.go:76-88` - `Read()` advances `readIdx` even when it returns an error such as `unknown tx hash in LedgerCloseMeta`

## Evidence

The local loops check only `if err == io.EOF { break }`, so any other `Read()` error falls through. The upstream reader increments its internal index before returning an error, so a failed read would be skipped rather than retried.

## Anti-Evidence

The concrete non-EOF reader errors I found all depend on malformed or inconsistent `LedgerCloseMeta` contents, or on upstream hashing support failing while ingesting envelopes. That requires broken ingest metadata or upstream SDK incompatibility rather than an ordinary legitimate Stellar transaction flowing through a healthy export path.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The control-flow bug is real, but the reachable triggers are malformed ledger-close metadata and upstream ingest-reader failures, which fall outside this objective's "legitimate transaction produces wrong export" scope.

### Lesson Learned

When a reader bug only activates after upstream ingestion has already produced inconsistent ledger metadata, treat it as an upstream or invalid-input boundary issue rather than a first-order external-io data-integrity finding.
