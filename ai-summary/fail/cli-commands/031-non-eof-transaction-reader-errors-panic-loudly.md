# H031: Non-EOF transaction-reader errors silently drop rows across archive exports

**Date**: 2026-04-14
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: partial exports from swallowed transaction-reader failures
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `txReader.Read()` returns a non-EOF error while an archive export is reading transactions, operations, or trades, the command should surface that error and abort cleanly rather than silently skipping or fabricating rows.

## Mechanism

`GetTransactions()`, `GetOperations()`, and `GetTrades()` only special-case `io.EOF`; they do not handle other `Read()` errors before using the returned `tx` value. At first glance this looks like a silent-omission path that could feed zero-value transactions into later transforms and leave a plausible-but-wrong export behind.

## Trigger

Force `txReader.Read()` to return a non-EOF error while running one of the archive export commands that depends on these readers.

## Target Code

- `internal/input/transactions.go:GetTransactions:50-62` — ignores non-EOF `txReader.Read()` errors
- `internal/input/operations.go:GetOperations:52-70` — same pattern
- `internal/input/trades.go:GetTrades:50-76` — same pattern
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/transaction_envelope.go:26-39` — zero-value `TransactionEnvelope.SourceAccount()` dereferences the TxV0 arm
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/transaction_envelope.go:226-236` — zero-value `TransactionEnvelope.Operations()` dereferences the TxV0 arm

## Evidence

The input loops do continue after non-EOF errors and immediately touch the returned `tx` value. The zero value of `xdr.TransactionEnvelope.Type` is `EnvelopeTypeEnvelopeTypeTxV0`, so the downstream methods are operating on a partially initialized union arm rather than an explicitly invalid sentinel.

## Anti-Evidence

Tracing the zero-value envelope semantics shows that the first downstream call is not a success-shaped omission: `TransactionEnvelope.Operations()` and related helpers dereference `e.V0`, which is nil on the zero value and panics. That turns this path into a loud crash rather than silent data corruption.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-14
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The swallowed-read-error path is real, but it does not yield a plausible partial export. The zero-value transaction envelope crashes when downstream code accesses operations or source-account data, so the failure is noisy and out of scope for this data-integrity objective.

### Lesson Learned

When a helper swallows an input-reader error, check the concrete zero-value semantics of the returned SDK type before assuming silent omission. Here, the XDR union's zero arm dereferences nil pointers quickly, converting the bug into a loud panic path.
