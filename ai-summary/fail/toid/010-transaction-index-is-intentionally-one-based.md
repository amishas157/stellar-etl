# H010: Transaction TOIDs do not currently suffer from a zero-vs-one-based transaction-order mismatch

**Date**: 2026-04-11
**Subsystem**: toid
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Every exported `transaction_id` / `operation_id` should use the same transaction-order convention across transactions, operations, contract events, token transfers, and effects. A first transaction in a ledger should not be encoded one way in some tables and a different way in others.

## Mechanism

The suspicion came from a conflict between the legacy `internal/toid` comment that says a ledger shares its id with its first transaction and the current upstream ingest contract that marks `LedgerTransaction.Index` as 1-indexed. That looked like a possible silent split where some exporters might still encode transaction order `0` while newer paths used `1`.

## Trigger

Compare the TOIDs emitted by `TransformTransaction`, `TransformOperation`, `TransformContractEvent`, `TransformTokenTransfer`, and `TransformEffect` for the first transaction in a ledger, then trace the upstream meaning of `LedgerTransaction.Index`.

## Target Code

- `internal/toid/main.go:51-59` — legacy package comment says a ledger shares an id with its first transaction.
- `internal/transform/transaction.go:25-28` — transaction export uses `transaction.Index` directly.
- `internal/transform/operation.go:31-32` — operation export uses the same transaction index plus `operationIndex+1`.
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction.go:15-18` — upstream ingest now documents `Index` as 1-indexed.
- `internal/transform/transaction_test.go:49-64,251-255` — current transaction tests expect transaction id `4096` when `Index == 1`.

## Evidence

There is a real documentation split: the local toid package still describes the older zero-based transaction-order convention, while modern ingest explicitly constructs transaction indices starting at `1`. Several test helpers in the repo also still synthesize `LedgerTransaction{Index: 0}`, which makes the drift easy to suspect.

## Anti-Evidence

All live TOID-bearing export paths that consume real ingest transactions consistently use the upstream 1-indexed contract, and their tests agree on that encoding. I did not find a production exporter that still serializes transaction TOIDs with zero-based transaction order.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated in `toid`

### Why It Failed

The mismatch is in comments and some unit-test scaffolding, not in live exported data. Current production TOID-bearing exports all consume `ingest.LedgerTransaction.Index` as 1-indexed and therefore agree with each other on the emitted IDs.

### Lesson Learned

When TOID documentation and callers disagree, trust the current upstream ingest contract and then verify every production exporter against it. Outdated comments can make a real-looking hypothesis, but consistent live call sites mean there is no present-day corruption bug.
