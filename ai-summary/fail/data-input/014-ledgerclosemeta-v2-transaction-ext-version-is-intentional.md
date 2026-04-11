# H014: LedgerCloseMeta V2 synthetic transaction ext version is intentional

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: Medium
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `GetLedgers()` synthesizes a `historyarchive.Ledger` from `LedgerCloseMetaV2`, it should preserve the full generalized transaction set so `TransformLedger()` can compute correct transaction and operation counts for protocol-23+ ledgers.

## Mechanism

At first glance, `internal/input/ledgers.go` looks like it down-converts `LedgerCloseMetaV2` by forcing `TransactionHistoryEntryExt.V = 1`. If that were a lossy cast, generalized-transaction-set fields introduced after classic `TxSet` handling could be dropped and ledger counts would become wrong on newer ledgers.

## Trigger

Export ledgers from a range containing `LedgerCloseMetaV2` and inspect whether the synthetic `historyarchive.Ledger.Transaction.Ext` loses protocol-23+ transaction-set structure.

## Target Code

- `internal/input/ledgers.go:GetLedgers:50-57` — `LedgerCloseMetaV2` path sets `TransactionHistoryEntryExt.V = 1`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:15053-15089` — `TransactionHistoryEntryExt` only has discriminants `0` and `1`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:19855-19864` — `LedgerCloseMetaV2.TxSet` is itself a `GeneralizedTransactionSet`

## Evidence

The V2 branch in `GetLedgers()` does hard-code `ext.V = 1`, which initially looks like a version mismatch against `LedgerCloseMeta.V = 2`.

## Anti-Evidence

The nested XDR definitions show that `TransactionHistoryEntryExt` is a separate union whose generalized-transaction-set arm is version `1`; the actual protocol-specific structure lives inside the nested `GeneralizedTransactionSet`, not in the outer `TransactionHistoryEntryExt` discriminant.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

There is no lossy down-conversion here. `TransactionHistoryEntryExt` only supports `V=0` (classic `TxSet`) and `V=1` (generalized transaction set), so setting `V=1` for `LedgerCloseMetaV2` is the correct way to preserve the nested `GeneralizedTransactionSet`.

### Lesson Learned

When two adjacent XDR unions have different version spaces, do not assume their discriminants should numerically match. Check the nested type definition first: the outer history-entry ext chooses "classic vs generalized," while the inner generalized transaction set carries the protocol-era structure.
