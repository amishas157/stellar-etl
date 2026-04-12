# H023: Token-transfer `transaction_id` should add `+1` to `TransactionIndex`

**Date**: 2026-04-12
**Subsystem**: toid
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For every token-transfer row, `transaction_id` should equal the canonical TOID of the parent transaction, matching the `history_transactions.id` value exported for the same ledger transaction. If the upstream token-transfer metadata were zero-based, the first transaction in a ledger would need `+1` before TOID packing so it does not collapse onto the ledger-level ID slot.

## Mechanism

`transformEvents()` feeds `eventMeta.TransactionIndex` directly into `toid.New(...)` with no normalization. That looked suspicious because several neighboring TOID producers normalize local zero-based indexes with `+1`, so a zero-based token-transfer transaction index would silently shift every exported `transaction_id` one transaction backward.

## Trigger

Export token transfers for a ledger containing a Soroban token-transfer event in the first applied transaction, then compare the row's `transaction_id` against the canonical transaction TOID from `history_transactions`.

## Target Code

- `internal/transform/token_transfer.go:82-90` — constructs `transaction_id` directly from `eventMeta.TransactionIndex`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251210093830-27970a00c2d7/processors/token_transfer/token_transfer_event.go:89-103` — upstream `EventMeta` constructor defines the transaction-index contract
- `internal/transform/token_transfer_test.go:156-167` — local unit fixture uses `TransactionIndex: 1` for the first transaction TOID

## Evidence

The ETL does no local `+1` adjustment in `transformEvents()`, so correctness depends entirely on the upstream metadata contract. The upstream protobuf serialization test also contains a bare `TransactionIndex: 0` fixture, which initially made the field look ambiguous enough to warrant review.

## Anti-Evidence

The real upstream constructor `NewEventMetaFromTx` explicitly documents `TransactionIndex: tx.Index // The index in ingest.LedgerTransaction is already 1-indexed`, while only `OperationIndex` is normalized from zero-based to one-based in that function. The ETL's own token-transfer tests also model the first transaction with `TransactionIndex: 1`, consistent with the canonical transaction TOID convention.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The upstream token-transfer metadata already carries the parent transaction index in the one-based `ingest.LedgerTransaction.Index` convention, so `transformEvents()` is packing the correct transaction TOID today.

### Lesson Learned

Token-transfer metadata mixes two different index origins: `TransactionIndex` is already normalized upstream, while `OperationIndex` is normalized inside the upstream constructor from a local zero-based value. Future TOID audits on this path must verify each field independently instead of assuming the whole proto follows one indexing convention.
