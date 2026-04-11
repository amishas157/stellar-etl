# H033: Fee-bump `max_fee` truncates the outer fee-bump bid

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a fee-bump transaction, the export should preserve both fee concepts without truncation: the inner transaction fee should remain available where the schema expects the inner fee, and the outer fee-bump bid should be preserved in a field wide enough for the XDR `int64` value. A fee-bump row with an outer fee larger than `uint32` should not silently wrap or narrow that outer fee.

## Mechanism

The initial suspicion was that `TransactionOutput.MaxFee` is still `uint32`, so fee-bump transactions might narrow the outer fee-bump bid. If `max_fee` were populated from `FeeBumpFee()`, any outer Soroban fee above `2^32-1` would be corrupted.

## Trigger

Process a fee-bump Soroban transaction whose outer `FeeBumpFee()` exceeds `4294967295`, then compare the exported fee fields against the inner fee and outer fee-bump bid.

## Target Code

- `internal/transform/schema.go:TransactionOutput:47-67` - `MaxFee` is `uint32`, while `NewMaxFee` is `int64`.
- `internal/transform/transaction.go:TransformTransaction:40-42` - `max_fee` is populated from `transaction.Envelope.Fee()`.
- `internal/transform/transaction.go:TransformTransaction:283-301` - fee-bump rows additionally populate `FeeAccount`, `InnerTransactionHash`, and `NewMaxFee`.
- `internal/transform/transaction_test.go:114-145` - tests expect fee-bump rows with `MaxFee: 0` and `NewMaxFee: 7200`.
- `internal/transform/transaction_test.go:183-216` - tests expect fee-bump Soroban rows with `MaxFee: 38333` and `NewMaxFee: 38533`.

## Evidence

The type asymmetry is real: `NewMaxFee` was recently widened to `int64`, while `MaxFee` remains `uint32`. That looks like a classic narrowing bug at first glance, especially because fee-bump outer fees are `int64` in XDR.

## Anti-Evidence

`TransformTransaction()` does not populate `MaxFee` from the outer fee-bump bid. It uses `transaction.Envelope.Fee()`, and the checked-in tests explicitly anchor fee-bump behavior where `MaxFee` carries the inner fee while `NewMaxFee` carries the outer fee-bump amount.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The code intentionally splits fee-bump fees across two fields: `max_fee` comes from the inner transaction fee, and `new_max_fee` carries the outer fee-bump bid. The apparent uint32 narrowing on `MaxFee` does not touch the outer `FeeBumpFee()` path that matters for overflow.

### Lesson Learned

In this package, similar fee names can refer to different layers of a fee-bump envelope. Before flagging a type mismatch, confirm which helper (`Fee()` vs `FeeBumpFee()`) actually feeds each exported column and cross-check the existing transaction tests for the intended split.
