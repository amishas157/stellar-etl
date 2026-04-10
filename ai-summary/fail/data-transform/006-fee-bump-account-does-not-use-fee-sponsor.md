# H006: Fee-bump `account` field is overwritten by the fee sponsor

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For fee-bump transactions, `history_transactions.account` should identify the inner transaction source account, while `fee_account` should identify the outer fee-bump sponsor. The two fields should diverge whenever one account pays fees for another account's inner transaction.

## Mechanism

At first glance, `TransformTransaction()` looks suspicious because it reads `transaction.Envelope.SourceAccount()` before the fee-bump-specific block sets `FeeAccount`. If `Envelope.SourceAccount()` resolved to the outer fee-bump envelope source, the exporter would write the fee sponsor into both `account` and `fee_account`, losing the inner transaction source.

## Trigger

Export a fee-bump transaction where the fee sponsor differs from the inner transaction source and compare `account` to `fee_account`.

## Target Code

- `internal/transform/transaction.go:29-30` — `account` is sourced from `transaction.Envelope.SourceAccount()`
- `internal/transform/transaction.go:284-291` — fee-bump block separately fills `fee_account`
- `internal/transform/transaction_test.go:122-135` — fee-bump fixture includes distinct expected `Account` and `FeeAccount`

## Evidence

The transform populates `Account` before the fee-bump-specific branch runs, which made it plausible that fee-bump transactions could misattribute the source account. The later block then assigns `FeeAccount` from `transaction.Envelope.FeeBumpAccount()`, so a wrong `SourceAccount()` interpretation would silently duplicate the fee sponsor across both fields.

## Anti-Evidence

The checked-in fee-bump test fixture expects `Account: testAccount1Address` and `FeeAccount: testAccount5Address`, showing that `transaction.Envelope.SourceAccount()` already resolves to the inner transaction source in this SDK abstraction.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The current SDK's `Envelope.SourceAccount()` behavior already preserves the inner transaction source for fee-bump envelopes, so `TransformTransaction()` exports distinct `account` and `fee_account` values as intended.

### Lesson Learned

When fee-bump transforms read `Envelope.SourceAccount()` before their fee-bump-specific branch, verify the SDK helper's semantics before assuming the outer fee sponsor leaked into the shared path.
