# H013: Soroban `inclusion_fee_charged` already isolates the charged inclusion fee

**Date**: 2026-04-15
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a Soroban transaction whose maximum inclusion-fee bid exceeds the network-clearing inclusion fee, `history_transactions.inclusion_fee_charged` should export the actual inclusion fee charged, not the submitter's higher bid.

## Mechanism

`TransformTransaction()` computes `inclusion_fee_charged` from the fee-account balance delta in `FeeChanges` and subtracts `sorobanData.resourceFee`. At first glance this looked like it would recover the declared inclusion-fee bid instead of the actual charged inclusion fee, because the envelope stores only the bid and Soroban transactions can overbid for inclusion.

## Trigger

Export any successful Soroban transaction whose inclusion-fee bid is higher than the inclusion fee ultimately required for ledger inclusion.

## Target Code

- `internal/transform/transaction.go:161-179` — derives `inclusion_fee_charged` from `FeeChanges`
- `github.com/stellar/go-stellar-sdk/ingest/ledger_transaction.go:540-555` — SDK helper uses the same formula
- `github.com/stellar/go-stellar-sdk/xdr/xdr_generated.go:17326-17335` — XDR comment defining the charged-fee relationship

## Evidence

The transformer does not derive `inclusion_fee_charged` from `result.feeCharged` or from the Soroban meta fee-breakdown fields. Instead it computes `initialFeeCharged := accountBalanceStart - accountBalanceEnd` from `transaction.FeeChanges`, then exports `initialFeeCharged - outputResourceFee`.

## Anti-Evidence

The official Stellar fee docs state the inclusion fee is a maximum willingness-to-pay and that only the necessary inclusion fee is collected. The SDK's `LedgerTransaction.SorobanInclusionFeeCharged()` helper deliberately uses the same `FeeChanges - resourceFee` formula, which strongly suggests `FeeChanges` already reflect the charged inclusion-fee component plus the reserved resource fee rather than the full bid.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The code path is consistent with the current protocol model: `FeeChanges` appear to record the actually charged inclusion fee together with the reserved resource fee, while post-apply fee changes cover resource-fee refunds. I did not find evidence that an inclusion-fee overbid creates an additional refund path that the transform misses.

### Lesson Learned

For Soroban fee fields, a suspicious hand-rolled calculation is not enough by itself. If the same formula is codified in the SDK helper layer and the protocol docs describe the envelope fee as a cap rather than an amount always deducted, treat the transform as following the current contract unless you can prove a concrete refund path the exporter ignores.
