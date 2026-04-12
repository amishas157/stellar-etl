# H008: Fee-Bump Outer and Inner Hashes Use Distinct XDR Fields

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Low
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For fee-bump transactions, `transaction_hash` should identify the outer fee-bump
envelope while `inner_transaction_hash` should identify the wrapped inner
transaction. The exporter should only be flagged if both fields are sourced
from the same underlying XDR hash.

## Mechanism

The fee-bump test fixture in `transaction_test.go` shows identical values for
`transaction_hash` and `inner_transaction_hash`, which initially suggested that
the ETL might be exporting the inner hash twice.

## Trigger

1. Export any fee-bump transaction.
2. Compare `history_transactions.transaction_hash` with
   `history_transactions.inner_transaction_hash`.
3. Suspect a bug if both fields always match.

## Target Code

- `internal/transform/transaction.go:22` — `transaction_hash` comes from
  `transaction.Result.TransactionHash`
- `internal/transform/transaction.go:292-293` — `inner_transaction_hash` comes
  from `transaction.Result.InnerHash()`
- `.../go-stellar-sdk/xdr/xdr_generated.go:14891-14894` — `TransactionResultPair`
  stores `TransactionHash` separately from `Result`
- `.../go-stellar-sdk/xdr/transaction_result.go:36-40` — `InnerHash()` reads
  the nested inner-result hash

## Evidence

The synthetic fee-bump test output hard-codes the same hex string for both
fields, which makes the mapping look suspicious at a glance.

## Anti-Evidence

The live code reads the two values from distinct XDR locations, and the SDK's
`InnerHash()` helper explicitly returns the nested inner-result hash rather than
the outer `TransactionResultPair.TransactionHash`.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The apparent collision comes from the synthetic unit fixture, not from the
transform logic. The ETL already uses separate XDR fields for outer and inner
hashes, so I do not have evidence of a real export-path corruption bug.

### Lesson Learned

When fixtures collapse two values to the same placeholder, verify the live XDR
source fields before filing a hash-mapping bug. Distinct helper methods backed
by distinct XDR fields are strong anti-evidence.
