# H011: Fee-bump transactions do not mis-export `max_fee`

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For fee-bump transactions, the export should distinguish the inner transaction fee bid from the outer fee-bump wrapper fee. The inner fee should stay in `max_fee`, while the outer fee-bump total should appear in `new_max_fee`, so downstream fee analysis can recover both values without ambiguity.

## Mechanism

At first glance, `TransformTransaction()` looks suspicious because it assigns `MaxFee` from `transaction.Envelope.Fee()` even on fee-bump envelopes, then separately fills `NewMaxFee` from `FeeBumpFee()`. If `Fee()` were already the outer wrapper fee, the export would duplicate or mislabel the two fee fields and corrupt fee-bump analytics.

## Trigger

Inspect any fee-bump transaction where the inner transaction fee and outer wrapper fee differ, such as the fee-bump fixtures in `transaction_test.go`.

## Target Code

- `internal/transform/transaction.go:40-45` — initializes `MaxFee` from `transaction.Envelope.Fee()`
- `internal/transform/transaction.go:283-295` — fills `FeeAccount`, `InnerTransactionHash`, and `NewMaxFee` for fee-bump envelopes
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/transaction_envelope.go:13-16` — defines `FeeBumpFee()` as the outer fee-bump fee
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/transaction_envelope.go:51-58` — defines `Fee()` as the inner transaction fee for fee-bump envelopes
- `internal/transform/transaction_test.go:114-146` — expects `MaxFee=0` and `NewMaxFee=7200` for a fee-bump fixture

## Evidence

The two exported columns use similar names, and fee-bump transactions are one of the few places where Stellar exposes both an inner and outer fee amount. Without checking the SDK helper semantics, the split can look like a mislabeled copy-paste.

## Anti-Evidence

The SDK explicitly documents that `TransactionEnvelope.Fee()` returns the inner fee for fee-bump envelopes, while `FeeBumpFee()` returns the outer fee-bump fee. The existing tests also assert the split behavior directly.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

`TransformTransaction()` is already using the two helper methods exactly as intended: `MaxFee` captures the inner transaction fee bid, and `NewMaxFee` captures the outer fee-bump fee. The suspected corruption disappears once the SDK helper semantics are traced.

### Lesson Learned

For fee-bump transactions, `Fee()` and `FeeBumpFee()` intentionally refer to different fee layers. Any future hypothesis in this area must verify the helper contract in `transaction_envelope.go` before treating similarly named fee fields as duplicates.
