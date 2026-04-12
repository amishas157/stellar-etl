# H015: Asset readers might drop fee-bump-wrapped payment/manage-sell operations

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `export_assets` includes `payment` and `manage_sell_offer` operations in its discovery scope, fee-bump-wrapped instances of those same inner operations should be discoverable too. A fee-bump envelope should not make otherwise in-scope asset rows disappear from the export.

## Mechanism

The asset readers iterate raw `xdr.TransactionEnvelope` values from `ledger.TransactionEnvelopes()` / `transform.GetTransactionSet(...)` instead of the normalized `LedgerTransactionReader` path used by sibling transaction exporters. That initially suggested the readers might only see the outer fee-bump wrapper and therefore miss inner payment or manage-sell operations entirely.

## Trigger

1. Pick a ledger range containing a fee-bump transaction whose inner transaction has a `payment` or `manage_sell_offer`.
2. Run `stellar-etl export_assets --start-ledger <S> --end-ledger <E>`.
3. Compare the resulting asset rows against the inner operation’s assets.

## Target Code

- `internal/input/assets.go:GetPaymentOperations:38-49` — scans `ledger.TransactionEnvelopes()` directly rather than `LedgerTransactionReader.Read()`.
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:28-39` — scans `transform.GetTransactionSet(ledger)` directly.
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/transaction_envelope.go:226-237` — upstream `TransactionEnvelope.Operations()` implementation.

## Evidence

Unlike `GetTransactions()`, `GetOperations()`, and `GetTrades()`, the asset readers bypass ingest’s normalized transaction reader and work straight from envelope helpers. That made fee-bump handling look suspicious because the command never explicitly unwraps `FeeBump.Tx.InnerTx` on its own.

## Anti-Evidence

The upstream SDK already performs the unwrap in the exact helper the readers call: `TransactionEnvelope.Operations()` switches on `EnvelopeTypeEnvelopeTypeTxFeeBump` and returns `e.FeeBump.Tx.InnerTx.V1.Tx.Operations`. Because both asset readers use that helper, fee-bump-wrapped payment/manage-sell operations remain visible to the export path.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected omission is prevented by the SDK contract the readers already rely on. `xdr.TransactionEnvelope.Operations()` unwraps fee-bump envelopes to the inner transaction’s operations, so the asset readers do not need a separate fee-bump special case to see in-scope operations.

### Lesson Learned

When a reader works directly from `xdr.TransactionEnvelope`, inspect the SDK envelope helpers before assuming wrapper types are opaque. In this codebase, `Operations()` and related helpers already normalize fee-bump envelopes, so “raw envelope scan” is not automatically a fee-bump omission bug.
