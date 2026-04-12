# H007: Fee-bump `max_fee` truncates the outer fee to `uint32`

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `history_transactions.max_fee` is supposed to carry the outer fee-bump fee, then fee-bump transactions whose outer fee exceeds `uint32` should still export the exact on-chain amount without truncation. The JSON and Parquet outputs should preserve the same fee value that appears in the fee-bump envelope.

## Mechanism

I investigated whether `TransactionOutput.MaxFee uint32` was incorrectly narrowing fee-bump fees that are exposed as `int64` on the outer envelope. If true, Soroban fee-bump transactions could silently export a wrapped or truncated `max_fee`.

## Trigger

Export a fee-bump Soroban transaction with a large outer fee and compare the exported `max_fee` with the envelope's fee-bump fee.

## Target Code

- `internal/transform/transaction.go:40` — populates `outputMaxFee` from `transaction.Envelope.Fee()`
- `internal/transform/transaction.go:239` — writes that value into `TransactionOutput.MaxFee`
- `internal/transform/transaction.go:283-295` — separately exports the outer fee-bump amount as `NewMaxFee`
- `internal/transform/transaction_test.go:114-145` — existing fee-bump fixture expects `MaxFee=0` and `NewMaxFee=7200`

## Evidence

The output schema uses `uint32` for `max_fee` even though fee-bump envelopes also expose an outer fee through `FeeBumpFee() int64`, so this initially looked like a classic narrowing bug.

## Anti-Evidence

The transform explicitly splits the two concepts: `MaxFee` is sourced from `Envelope.Fee()`, which the SDK defines as the inner transaction fee for fee-bump envelopes, while `NewMaxFee` carries the outer fee-bump fee. The tests already assert this exact split for fee-bump transactions.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This is an intentional schema split, not truncation. The exporter already preserves the outer fee-bump amount in `new_max_fee`, so the apparent `uint32` narrowing on `max_fee` does not lose the fee-bump value being exported.

### Lesson Learned

When auditing fee-bump fields, verify the semantic contract of each column before treating type differences as corruption. In this schema, `max_fee` and `new_max_fee` intentionally represent different fee layers.
